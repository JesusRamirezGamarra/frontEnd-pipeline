pipeline {
    agent any
    environment {
        VERCEL_TOKEN = credentials('VERCEL_TOKEN')
    }

    stages {
        stage ('Instalar dependencias...') {
            agent {
                docker { image 'node:18-alpine'}
            }
            steps {
                echo "Remover dependencias antiguas o referencias por el json.lock"
                sh 'rm -rf node_modules package-lock.json'
                sh 'npm install'
            }
        }

        stage ('Construir proyecto con archivos estaticos...') {
            agent {
                docker { image 'node:18-alpine'}
            }
            steps {
                sh 'npm run build'
            }
        }

        stage('Subir archivos al bucket de respaldo ...') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: 'us-east-1') {
                    script {
                        def ultimaCarpetaDeBackup = sh(returnStdout: true, script: '''
                            aws s3 ls s3://bucket-codigo-backup/JesusRamirez/vercel/ | awk '{print $2}' | grep VERSION_ | sort | tail -n 1
                        ''').trim()

                        echo "Ultima carpeta del bucket backup: ${ultimaCarpetaDeBackup}"

                        def baseVersion = 'VERSION_1.0'

                        if (ultimaCarpetaDeBackup) {
                            def currentVersion = ultimaCarpetaDeBackup.replace('VERSION_','').replace('/', '')

                            echo "Version actual: ${currentVersion}"

                            def versionNumber = currentVersion.toFloat() + 0.1

                            echo "Numero de Version aumentado : ${versionNumber}"

                            baseVersion = String.format("VERSION_%.1f", versionNumber)

                            echo "Nombre de version formateado : ${baseVersion}"
                        }

                        echo "Subiendo los archivos al bucket s3 en la carpeta ${baseVersion}..."
                        sh """
                            aws s3 sync build/ s3://bucket-codigo-backup/JesusRamirez/vercel/${baseVersion}/ --delete
                        """
                    }                   
                }
            }
        }

        stage ('Deploy hacia Vercel...') {
            agent {
                docker { image 'node:18-alpine'}
            }
            steps {
                sh """
                    npm install -g vercel
                    vercel deploy --prod --name front-vercel --token $VERCEL_TOKEN --yes
                """
            }
        }
    }
}