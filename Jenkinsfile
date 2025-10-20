// Jenkinsfile (Declarative Pipeline) - Site HTML (pédagogique)
pipeline {
  agent any

  // Paramètres accessibles depuis l'interface Jenkins lorsque tu lances le job
  parameters {
    choice(name: 'DEPLOY_METHOD', choices: ['NO', 'SSH', 'GHPAGES'], description: 'Mode de déploiement : NO = ne pas déployer, SSH = rsync/ssh, GHPAGES = push sur gh-pages')
    string(name: 'SSH_DEST', defaultValue: 'user@serveur:/var/www/site', description: 'Destination rsync si SSH (user@host:/chemin)')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branche de code source')
  }

  environment {
    // Id des credentials dans Jenkins - à créer dans Jenkins > Credentials
    SSH_CREDENTIALS_ID = 'ssh-deploy-key-id'           // pour sshagent (clé privée)
    GITHUB_CREDENTIALS_ID = 'github-user-token'        // username/password or token for git push
    NODEJS_TOOL = 'node16'                             // optionnel : nom d'installation NodeJS configurée dans Jenkins Global Tools
    BUILD_DIR = 'dist'                                 // dossier de sortie construit (par défaut dist). Change si besoin.
  }

  options {
    // Nettoie le workspace après exécution (facultatif)
    buildDiscarder(logRotator(numToKeepStr: '10'))
    ansiColor('xterm') // couleur dans la console (si plugin installé)
    timestamps()
  }

  stages {

    stage('Checkout') {
      steps {
        script {
          // Checkout de la branche spécifiée par le paramètre
          checkout([
            $class: 'GitSCM',
            branches: [[name: "refs/heads/${params.GIT_BRANCH}"]],
            userRemoteConfigs: [[url: env.GIT_URL ?: scm.userRemoteConfigs[0].url]]
          ])
        }
      }
    }

    stage('Detect & Setup') {
      steps {
        script {
          // Détection simple : y a-t-il un package.json ? (projet node)
          hasPackageJson = fileExists('package.json')
          echo "Présence de package.json : ${hasPackageJson}"
        }
      }
    }

    stage('Install dependencies (si nécessaire)') {
      when {
        expression { return hasPackageJson == true }
      }
      tools {
        // active Node.js si configuré globalement dans Jenkins (optionnel)
        nodejs "${env.NODEJS_TOOL}"
      }
      steps {
        echo 'Installation des dépendances npm...'
        sh 'npm ci' // npm ci est plus reproductible pour CI si package-lock.json présent
      }
    }

    stage('Lint / Check HTML') {
      steps {
        script {
          if (hasPackageJson) {
            // Si le projet a des scripts npm pour le linting, on les utilise
            def hasLintScript = false
            if (fileExists('package.json')) {
              def json = readFile('package.json')
              hasLintScript = json.contains('"lint"') || json.contains('"htmlhint"')
            }
            if (hasLintScript) {
              echo "Lancement du script npm lint"
              sh 'npm run lint || true' // ne casse pas forcément si on veut warnings
            } else {
              echo "Aucun script lint configuré dans package.json — on fera une vérification basique."
              // Exemple simple : vérifier qu'il existe au moins index.html
              if (!fileExists('index.html') && !fileExists('src/index.html')) {
                error "Impossible de trouver index.html — arrête la build."
              } else {
                echo "index.html présent — check basique OK."
              }
            }
          } else {
            // Projet pur HTML (pas de node)
            echo "Pas de package.json — projet statique HTML simple."
            if (!fileExists('index.html')) {
              error "index.html manquant dans la racine du repo."
            }
          }
        }
      }
    }

    stage('Build (optionnel)') {
      steps {
        script {
          // Si le repo a un script build dans package.json, on l'utilise
          def needsBuild = false
          if (hasPackageJson) {
            def json = readFile('package.json')
            needsBuild = json.contains('"build"')
          }
          if (needsBuild) {
            echo "Exécution de npm run build..."
            sh 'npm run build'
          } else {
            echo "Aucun build step détecté — on prépare les fichiers statiques tels quels."
            // Copier les fichiers vers BUILD_DIR pour uniformiser
            sh "rm -rf ${env.BUILD_DIR} || true"
            sh "mkdir -p ${env.BUILD_DIR}"
            // copier tout sauf .git et node_modules
            sh "rsync -a --exclude='.git' --exclude='node_modules' ./ ${env.BUILD_DIR}/"
          }
        }
      }
    }

    stage('Archive artifact') {
      steps {
        script {
          // Archive les fichiers construits pour consultation et téléchargement
          archiveArtifacts artifacts: "${env.BUILD_DIR}/**/*", allowEmptyArchive: false
          // Stash pour réutilisation dans la phase de déploiement sur un autre agent
          stash includes: "${env.BUILD_DIR}/**/*", name: 'site-artifact'
        }
      }
    }

    stage('Deploy (optionnel)') {
      when {
        expression { return params.DEPLOY_METHOD != 'NO' }
      }
      steps {
        script {
          if (params.DEPLOY_METHOD == 'SSH') {
            echo "Déploiement via SSH/rsync vers ${params.SSH_DEST}"

            // Récupère l'artifact
            unstash 'site-artifact'

            // Utilise l'identifiant de credentials SSH configuré dans Jenkins (ssh-agent plugin requis)
            sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
              // Exemple : utiliser rsync pour transférer le contenu du BUILD_DIR
              sh """
                echo "Lancement de rsync vers ${params.SSH_DEST} ..."
                rsync -avz --delete ${env.BUILD_DIR}/ ${params.SSH_DEST}
              """
            }
          } else if (params.DEPLOY_METHOD == 'GHPAGES') {
            echo "Déploiement sur GitHub Pages (branche gh-pages)"

            // Récupère l'artifact
            unstash 'site-artifact'

            // Configure git et push sur gh-pages.
            // Nécessite des credentials (token) stockés en tant que usernamePassword dans Jenkins.
            withCredentials([usernamePassword(credentialsId: env.GITHUB_CREDENTIALS_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
              sh """
                git config user.email "jenkins@ci.local"
                git config user.name "Jenkins CI"
                # init temporaire
                cd ${env.BUILD_DIR}
                git init
                git remote add origin ${scm.userRemoteConfigs[0].url}
                git checkout -b gh-pages
                git add --all
                git commit -m "Deploy from Jenkins: ${env.BUILD_TAG}"
                # push via token (attention à l'URL: https://token@github.com/owner/repo.git)
                git push https://${GIT_USER}:${GIT_TOKEN}@${scm.userRemoteConfigs[0].url.replaceAll('https://','')} --force gh-pages
              """
            }
          } else {
            echo "Méthode de déploiement inconnue : ${params.DEPLOY_METHOD}"
          }
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline terminé avec succès ✅"
      // tu peux ajouter des notifications (email, slack) ici si souhaité
    }
    failure {
      echo "Le pipeline a échoué ❌"
      // Notifications d'échec possibles
    }
    always {
      cleanWs() // nettoie le workspace (nécessite plugin Workspace Cleanup)
    }
  }
}
