pipeline {
    agent any

    environment {
        SECRET_ENV = credentials('futurekawa.env')
         GITHUB_TOKEN = credentials('github-api-token')
    }

    triggers {
         pollSCM('H H/12 * * *')
    }

        stages {

            stage('1. Récupération du code') {
                steps {
                    echo '=== CLONAGE DES DIFFÉRENTS DEPOS GIT ==='

                    dir('mspr_bloc4') {  
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/master']],
                            userRemoteConfigs: [[
                                url: 'https://github.com/Alextraterrestre/mspr_bloc4.git',
                                credentialsId: 'git-tok'
                            ]]
                        ])
                    }
                    
                    dir('futureKawaFront') {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],
                            userRemoteConfigs: [[url: 'https://github.com/loanth/futureKawaFront']]
                        ])
                    }

                    dir('futurekawa') {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],
                            userRemoteConfigs: [[url: 'https://github.com/quentinchad/futurekawa']]
                        ])
                    }
                    
                    dir('MSPR1') {
                        echo 'Téléchargement de la Branche de test de l\'API...'
                         checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/branche_de_test']],
                            userRemoteConfigs: [[url: 'https://github.com/Luteix/MSPR1']]
                        ])
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

            stage('3.2 vérification du code front (ESLint)') {
                steps {
                    dir('futureKawaFront') {
                         script {
                            sh 'docker build -f Dockerfile -t futurekawa-front-lint .'
                            sh 'docker run --rm futurekawa-front-lint npm run lint'

                            echo '=== LINTER ==='
                            def lintResult = sh(
                                script: 'docker run --rm futurekawa-front-lint:${BUILD_NUMBER} npm run lint 2>&1 | tee lint-report.log',
                                returnStatus: true
                            )

                            echo '=== LOGS DU LINTER ==='
                            sh 'cat lint-report.log'

                            if (lintResult == 0) {
                                echo 'LINT VALIDÉ : Le code front respecte les règles ESLint.'
                            } else {
                                error "LINT EN ERREUR : Des erreurs ESLint ont été détectées (code retour ${lintResult}). Voir les logs ci-dessus."
                            }
                        }
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
