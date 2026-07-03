pipeline {
    agent any

    environment {
        SECRET_ENV = credentials('futurkawa.env')
    }

        stages {
            stage('1. Phase de Test : Récupération du code (branche_de_test)') {
                steps {
                    echo '=== CLONAGE DE LA BRANCHE DE TEST ==='
                    
                    dir('futureKawaFront') { git url: 'https://github.com/loanth/futureKawaFront', branch: 'main' }
                    dir('futurekawa') { git url: 'https://github.com/quentinchad/futurekawa', branch: 'main' }
                    
                    // On clone l'API sur la branche de test directement dans le dossier MSPR1
                    dir('MSPR1') {
                        echo 'Téléchargement de l\'API (Branche de test)...'
                        git url: 'https://github.com/Luteix/MSPR1', branch: 'branche_de_test'
                    }
                }
            }

            stage("2. Phase de Test : Configuration et Build") {
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

            stage("3. Phase de Test : Exécution de Pytest") {
                steps {
                    script {
                        echo '=== LANCEMENT DES TESTS UNITAIRES ==='
                        // On exécute pytest directement. Les logs seront entièrement visibles dans la console Jenkins.
                        // L'option -s permet de forcer l'affichage des prints s'il y en a.
                        def testResult = sh script: 'docker compose run --rm web sh -c "pytest -v -s"', returnStatus: true
                        
                        if (testResult != 0) {
                            error "Les tests unitaires ont échoué avec le code ${testResult}. Le pipeline s'arrête ici."
                        }
                        echo '=== TESTS VALIDÉS AVEC SUCCÈS ==='
                    }
                }
            }

            stage("4. Phase de Prod : Nettoyage du Workspace de Test") {
                steps {
                    echo 'Nettoyage du code de test pour préparer le déploiement propre de la production...'
                    // On supprime proprement le dossier MSPR1 qui contenait la branche de test
                    sh 'rm -rf MSPR1'
                    // On nettoie les conteneurs de test pour éviter les conflits de cache
                    sh 'docker compose down --v --remove-orphans'
                }
            }

            stage("5. Phase de Prod : Récupération de la branche Main") {
                steps {
                    echo '=== CLONAGE DE LA BRANCHE MAIN (PRODUCTION) ==='
                    dir('MSPR1') {
                        echo 'Téléchargement du code officiel de l\'API...'
                        git url: 'https://github.com/Luteix/MSPR1', branch: 'main'
                    }
                    // On s'assure que le .env est toujours présent pour la prod
                    sh 'cp $SECRET_ENV .env'
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
}