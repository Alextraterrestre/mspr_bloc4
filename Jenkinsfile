pipeline {
    agent any

    environment {
        SECRET_ENV = credentials('futurkawa.env')
    }

    stages {

        stage('1. Récupération des différents Dépôts') {
            steps {
                echo 'Téléchargement des différents dépôt Github'
                
                dir('futureKawaFront') {
                    echo 'Téléchargement du dépôt Frontend...'
                    git url: 'https://github.com/loanth/futureKawaFront', branch: 'main'
                }
                
                dir('MSPR1') {
                    echo "Téléchargement du dépôt de l'API..."
                    git url: 'https://github.com/Luteix/MSPR1', branch: 'main'
                }

                dir('MSPR1_test') {
                    echo 'Téléchargement du dépôt API (branche de tests)...'
                    git url: 'https://github.com/Luteix/MSPR1', branch: 'branche_de_test'
                }

                dir('futurekawa') {
                    echo 'Téléchargement du dépôt IoT...'
                    git url: 'https://github.com/quentinchad/futurekawa', branch: 'main'
                }
            }
        }

        stage("2. Injection de la configuration sécurisée") {
            steps {
                echo 'Mise en place du fichier .env et de config.py'
                sh 'cp $SECRET_ENV .env'
            }
        }

        stage("3. Build de l\'infrastructure") {
            steps {
                echo 'Compilation des images Docker (Flask, Front, Bases)...'
                retry(3) {
                    sh 'docker compose build db'
                    sh 'docker compose build mosquitto'
                    sh 'docker compose build web'
                    sh 'docker compose build front'
                    sh 'docker compose build db_iot'
                    sh 'docker compose build flask'
                }
            }
        }

        stage("4. Tests : Tests unitaires") {
            steps {
                script {
                    echo 'Exécution des tests sur la branche_de_test via sur-encrement de volume...'
                    // On monte le dossier MSPR1_test sur /app et on force le PYTHONPATH
                    def testResult = sh script: '''
                        docker compose run --rm \
                        -v "$(pwd)/MSPR1_test:/app" \
                        web sh -c "cd /app && PYTHONPATH=/app pytest -v /app/test_unitaire"
                    ''', returnStatus: true
                    
                    if (testResult != 0) {
                        error "Les tests unitaires sur la branche_de_test ont échoué (code d'erreur: ${testResult}). Déploiement annulé."
                    }
                    echo 'Tests validés avec succès ! Prêt pour le déploiement de main.'
                }
            }
        }
        // stage("4. Tests : Tests unitaires") {
        //     steps {
        //         script {
        //             sh 'echo "=== DEBUG: contenu de MSPR1_test ===" && ls -la $WORKSPACE/MSPR1_test/ && ls -la $WORKSPACE/MSPR1_test/*/'
        //             sh 'find $WORKSPACE/MSPR1_test/ -maxdepth 2 -type f | head -20'
                    
        //             def testResult = sh script: '''
        //                 docker compose run --rm -v "$(pwd)/MSPR1_test:/app" web sh -c "
        //                     cd /app && ls -la && find /app -type f | head -30
        //                 "
        //             ''', returnStatus: true
        //         }
        //     }
        // }

        stage("5. Lancement de l'application") {
            steps {
                echo 'Démarrage de l\'environnement de production...'
                sh 'docker compose up -d'

                echo 'Attente de la stabilité des conteneurs...'
                sh 'sleep 10'
            }
        }

        stage('6. Nettoyage') {
            steps {
                echo 'Nettoyage de l\'espace de build...'
                sh 'docker image prune -f'
            }
        }
    }

    post {
        always {
            echo 'Suppression sécurisée du fichier .env du workspace...'
            sh 'rm -f .env'
        }
        success {
            echo 'MSPR1 : Pipeline deployée ! Le code est stable et les conteneurs compilent.'
        }
        failure {
            echo 'Erreur sur la pipeline. Vérifier les logs.'
        }
    }
}
