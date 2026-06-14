pipeline {
    agent any

    stages {
        stage('1. Récupération des différents Dépôts') {
            steps {
                echo 'Téléchargement du Backend principal...'
                // Le dépôt principal (où se trouve le Jenkinsfile) se télécharge tout seul
                
                echo 'Téléchargement du dépôt Frontend...'
                dir('futureKawaFront') {
                    git url: 'https://github.com/loanth/futureKawaFront', branch: 'main'
                }

                echo "Téléchargement du dépôt de l'API..."
                dir('futurekawapp') {
                    git url: 'https://github.com/Luteix/MSPR1', branch: 'main'
                }

                echo 'Téléchargement du dépôt IoT...'
                dir('futurekawa') {
                    git url: 'https://github.com/quentinchad/futurekawa'
            }
        }

        stage('2. Injection de la configuration sécurisée') {
            steps {
                echo '=== ÉTAPE 4 : Création du fichier config.py à la volée ==='
                // On recrée le fameux fichier config.py qu'on a corrigé ensemble 
                // pour que l'API Flask ne crash pas au démarrage !
                dir('MSPR1') {
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

        stage('3. Build de l\'infrastructure') {
            steps {
                echo 'Compilation des images Docker (Flask, Front, Bases)...'
                # Le --no-cache force à tout reconstruire proprement pour le test
                sh 'docker compose build --no-cache'
            }
        }

        stage('4. Tests et Lancement') {
            steps {
                echo 'Démarrage de l\'environnement de test...'
                sh 'docker compose up -d'
                echo 'Attente de la stabilité des conteneurs...'
                sh 'sleep 10'
                sh 'docker compose ps'
            }
        }
        
        stage('5. Nettoyage') {
            steps {
                echo 'Nettoyage de l\'espace de build...'
                # On éteint après validation pour ne pas bloquer les ressources
                sh 'docker compose down'
            }
        }
    }

    post {
        success {
            echo '🎉 MSPR1 : Pipeline réussi ! Le code est stable et les conteneurs compilent.'
        }
        failure {
            echo '❌ Erreur sur le pipeline. Vérifie les logs de build.'
        }
    }
}