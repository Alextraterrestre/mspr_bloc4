pipeline {
    agent any

    environment {
        SECRET_ENV = credentials('futurekawa.env')
    }

        stages {
            
            stage('1. Récupération du code') {
                steps {
                    echo '=== CLONAGE DE LA BRANCHE DE TEST ==='
                    
                    dir('futureKawaFront') { git url: 'https://github.com/loanth/futureKawaFront', branch: 'main' }

                    dir('futurekawa') { git url: 'https://github.com/quentinchad/futurekawa', branch: 'main' }
                    
                    dir('MSPR1') {
                        echo 'Téléchargement de l\'API (Branche de test)...'
                        git url: 'https://github.com/Luteix/MSPR1', branch: 'branche_de_test'
                    }
                }
            }

            stage("2. Configuration et Build") {
                steps {
                    echo 'Mise en place de l\'environnement de test...'
                    sh 'cp $SECRET_ENV .env'
                    
                    echo 'Compilation des conteneurs pour exécuter les tests...'
                    retry(3) {
                        sh 'docker compose build db'
                        sh 'docker compose build web'
                    }
                }
            }

            stage("3. Phase de Test : Exécution des tests unitaires (pytest)") {
                steps {
                    script {
                        echo '=== LANCEMENT DES TESTS UNITAIRES ==='

                        def testResult = sh script: 'docker compose run --rm web sh -c "pytest -v -s"', returnStatus: true
                        if (testResult != 0) {
                            error "Les tests unitaires ont échoué avec le code ${testResult}. Le pipeline s'arrête ici."
                        }
                        echo '=== TESTS VALIDÉS AVEC SUCCÈS ! ==='
                    }
                }
            }

            stage("4. Nettoyage du Workspace de Test") {
                steps {
                    echo 'Nettoyage du code de test pour préparer le déploiement propre de la production...'
                    // suppression du dossier de la branche de tests
                    sh 'rm -rf MSPR1'
                    // On nettoie les conteneurs de test pour éviter les conflits de cache
                    sh 'docker compose down -v --remove-orphans'
                }
            }

            stage("5. Récupération de la branche Main") {
                steps {
                    echo '=== CLONAGE DE LA BRANCHE MAIN ==='
                    dir('MSPR1') {
                        echo 'Téléchargement du code officiel de l\'API...'
                        git url: 'https://github.com/Luteix/MSPR1', branch: 'main'
                    }
                    sh 'cp $SECRET_ENV .env'
                }
            }
            stage("6. Build & Déploiement de l'application futurekawa") {
                steps {
                    echo '=== DEPLOIEMENT DE LA PRODUCTION ==='
                    retry(3) {
                        sh 'docker compose build'
                    }
                    echo 'Démarrage officiel de l\'application en arrière-plan...'
                    sh 'docker compose up -d'
                    
                    sh 'sleep 10'
                    sh 'docker compose ps'
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
