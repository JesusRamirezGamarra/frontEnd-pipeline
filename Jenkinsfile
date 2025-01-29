pipeline {
    agent any

    environment {
        WEBHOOK_URL = 'https://c6c6-38-25-17-72.ngrok-free.app/github-webhook/' // URL del webhook
        AWS_REGION = 'us-east-1' // Región de AWS
        S3_BUCKET = 'bucket-codigo-jesus' // Nombre del bucket S3
        RECIPIENT_EMAIL = 'luciojesusramirezgamarra@gmail.com' // Dirección de correo electrónico del destinatario
        BACKUP_BUCKET = 'bucket-codigo-backup' // Bucket para el respaldo
    }

    stages {
        stage('Instalar dependencias...') {
            agent {
                docker { image 'node:16-alpine' }
            }
            steps {
                sh 'npm install'
            }
        }

        stage('Construir proyecto con archivos estáticos...') {
            agent {
                docker { image 'node:16-alpine' }
            }
            steps {
                sh 'npm run build'
            }
        }

        stage('Preparar estructura de buckets y realizar backup (solo main)') {
            when {
                branch 'main' // Ejecutar solo si la rama es `main`
            }
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: "${AWS_REGION}") {
                    script {
                        def ultimaCarpetaDeBackup = sh(returnStdout: true, script: '''
                            aws s3 ls s3://${BACKUP_BUCKET}/JesusRamirez/ | awk '{print $2}' | grep VERSION_ | sort | tail -n 1 || echo ""
                        ''').trim()
                        def baseVersion = 'VERSION_1.0'

                        if (ultimaCarpetaDeBackup) {
                            try {
                                def currentVersion = ultimaCarpetaDeBackup.replace('VERSION_', '').replace('/', '').toFloat()
                                def versionNumber = currentVersion + 0.1
                                baseVersion = String.format('VERSION_%.1f', versionNumber)
                                echo "Última versión encontrada: ${ultimaCarpetaDeBackup}. Nueva versión: ${baseVersion}"
                            } catch (Exception e) {
                                error "Error procesando la versión: ${e.message}"
                            }
                        } else {
                            echo "No se encontraron versiones previas en el bucket, usando la versión base: ${baseVersion}"
                        }

                        echo "Subiendo los archivos al bucket de backup en la carpeta: ${baseVersion}..."
                        sh '''
                            if [ -d "build/" ]; then
                                aws s3 sync build/ s3://${BACKUP_BUCKET}/JesusRamirez/${baseVersion} --delete
                            else
                                echo "Error: La carpeta 'build/' no existe o está vacía."
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }

        stage('Subir proyecto al bucket S3 de AWS ...') {
            when {
                not { branch 'develop' } // Ejecutar si NO es la rama `develop`
                not { branch 'QA' }      // También omitir si es la rama `QA`
                branch 'main'            // Ejecutar solo si la rama es `main`
            }
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: "${AWS_REGION}") {
                    script {
                        echo "Subiendo los archivos al bucket S3..."
                        sh """
                            aws s3 sync build/ s3://${S3_BUCKET} --delete
                        """
                    }
                }
            }
        }

        stage('Restaurar carpeta específica desde S3') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: "${AWS_REGION}") {
                    script {
                        def restoreVersion = 'VERSION_1.0.5' // Cambia esta versión según lo necesites
                        echo "Restaurando la carpeta ${restoreVersion} desde el bucket de backup..."
                        sh '''
                            mkdir -p restore/
                            aws s3 sync s3://${BACKUP_BUCKET}/JesusRamirez/${restoreVersion}/ ./restore/ --exact-timestamps
                        '''
                        echo "Archivos restaurados en el directorio './restore/'."
                    }
                }
            }
        }
    }

    post {
        success {
            mail to: 'luciojesusramirezgamarra@gmail.com',
                subject: "Pipeline ${env.JOB_NAME} ejecución correcta",
                body: """
                Hola,

                El pipeline '${env.JOB_NAME}' (Build #${env.BUILD_NUMBER}) ha finalizado de manera correcta.

                Los detalles se pueden revisar en el siguiente enlace:
                ${env.BUILD_URL}

                Saludos,
                Jenkins Server
                """
        }
        failure {
            mail to: 'luciojesusramirezgamarra@gmail.com',
                subject: "⚠️ Pipeline ${env.JOB_NAME} falló",
                body: """
                Hola,

                El pipeline '${env.JOB_NAME}' (Build #${env.BUILD_NUMBER}) ha fallado.

                Revisa los detalles del error en el siguiente enlace:
                ${env.BUILD_URL}

                Saludos,
                Jenkins Server
                """
        }
    }
}
