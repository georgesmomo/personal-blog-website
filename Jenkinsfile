// Jenkinsfile — exemple pédagogique (site HTML statique)
pipeline {
    agent any   // exécute sur n’importe quel agent (ton instance Jenkins locale par ex.)

    stages {

        stage('Préparation') {
            steps {
                echo "=== Étape 1 : Préparation du workspace ==="
                sh 'pwd'          // affiche le répertoire courant
                sh 'ls -la'       // liste les fichiers du workspace Jenkins
            }
        }

        stage('Build (simulation)') {
            steps {
                echo "=== Étape 2 : Construction du site (simulation) ==="
                // On simule la construction d’un site HTML statique
                sh '''
                    echo "Création du dossier de build..."
                    mkdir -p dist
                    echo "<h1>Bienvenue sur mon site HTML généré par Jenkins</h1>" > dist/index.html
                    echo "Build terminé !"
                '''
            }
        }

        stage('Tests (fictifs)') {
            steps {
                echo "=== Étape 3 : Tests ==="
                // On fait semblant de tester le site
                sh '''
                    echo "Vérification de la présence du fichier index.html..."
                    if [ -f dist/index.html ]; then
                        echo "✅ Test réussi : index.html existe."
                    else
                        echo "❌ Test échoué : index.html manquant."
                        exit 1
                    fi
                '''
            }
        }

        stage('Déploiement (simulation)') {
            steps {
                echo "=== Étape 4 : Déploiement fictif ==="
                // Aucune vraie connexion ici, juste une simulation
                sh '''
                    echo "Déploiement du site dans le dossier /var/www/html (fictif)"
                    echo "Copie en cours..."
                    sleep 2
                    echo "Déploiement terminé ! 🚀"
                '''
            }
        }

    }

    post {
        success {
            echo "Pipeline terminé avec succès ✅"
        }
        failure {
            echo "Pipeline échoué ❌"
        }
        always {
            echo "Nettoyage du workspace..."
            sh 'rm -rf dist || true'
        }
    }
}
