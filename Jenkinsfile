pipeline {
    agent any

    environment {
        SECRET_ENV = credentials('EnvFile')
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
                        git url: 'https://github.com/Luteix/MSPR1', branch: 'branche_de_tests'
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
                    dir("futurekawapp") {
                        sh '''
                        cat << 'EOF' > config.py
                        import os
                        
                        class Config:
                            """Configuration générée automatiquement par Jenkins"""
                            JWT_SECRET_KEY = os.getenv('JWT_SECRET_KEY', 'c0d46ff96e5372e4ea70aed573d1dba656b2564d340d9ba3ea33cbf4afe541b6')
                            JWT_ACCESS_TOKEN_EXPIRES = int(os.getenv('JWT_ACCESS_TOKEN_EXPIRES', 24))
                            JWT_ALGORITHM = os.getenv('JWT_ALGORITHM', 'HS256')
                            BCRYPT_ROUNDS = int(os.getenv('BCRYPT_ROUNDS', 12))
                            DB_USER = os.getenv('DB_USER', 'root')
                            DB_PASSWORD = os.getenv('DB_PASSWORD', 'secret')
                            DB_HOST = os.getenv('DB_HOST', 'db')
                            DB_NAME = os.getenv('DB_NAME', 'futurekawa')
                            SQLALCHEMY_DATABASE_URI = f"mysql+pymysql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}/{DB_NAME}"
                            SQLALCHEMY_TRACK_MODIFICATIONS = False

                        class DevelopmentConfig(Config):
                            DEBUG = True

                        class ProductionConfig(Config):
                            DEBUG = False
                            JWT_ACCESS_TOKEN_EXPIRES = 8

                        config = {
                            'development': DevelopmentConfig,
                            'production': ProductionConfig,
                            'default': DevelopmentConfig
                        }
                        EOF
                                        '''
                    }
                }
            }

            stage("3. Build de l\'infrastructure") {
                steps {
                    echo 'Compilation des images Docker (Flask, Front, Bases)...'
                    sh 'docker compose build --no-cache'
                }
            }

            stage("4. Tests : Tests unitaires") {
                steps {
                    script {
                        def testResult = sh script: 'docker compose run --rm -v "$(pwd)/MSPR1_test:/app" web sh -c "pip install pytest --break-system-packages && pytest"',
                        returnStatus: true
                        if (testResult != 0) {
                            error "Les tests unitaires ont échoué (code d'erreur: ${testResult})"
                        }
                    }
                }
            }

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
                // Le bloc post/always s'exécute QUOI QU'IL ARRIVE (succès ou échec des tests)
                // C'est l'endroit parfait pour supprimer le fichier .env et éviter les fuites de secrets
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
