pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1' // Región de AWS
        S3_BUCKET = 'bucket-codigo-jesus' // Bucket de destino final
        BACKUP_BUCKET = 'bucket-codigo-backup' // Bucket de backup
        RECIPIENT_EMAIL = 'luciojesusramirezgamarra@gmail.com' // Email de notificación
    }

    stages {
        stage('Instalar dependencias') {
            agent {
                docker { image 'node:16-alpine' }
            }
            steps {
                sh 'npm install'
            }
        }

        stage('Construir proyecto con archivos estáticos') {
            agent {
                docker { image 'node:16-alpine' }
            }
            steps {
                sh 'npm run build'
            }
        }

        stage('Backup y versión incremental en S3') {
            when {
                branch 'main' // Solo se ejecuta en la rama `main`
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
                        // Obtener la última versión en el bucket
                        def ultimaCarpetaDeBackup = sh(returnStdout: true, script: '''
                            aws s3 ls s3://${BACKUP_BUCKET}/JesusRamirez/ | awk '{print $2}' | grep VERSION_ | sort -V | tail -n 1 || echo ""
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

                        // Subir los archivos al bucket de backup con la nueva versión
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

        stage('Subir proyecto final al bucket S3 de AWS') {
            when {
                branch 'main' // Solo en `main`
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
                        echo "Subiendo los archivos al bucket final en S3..."
                        sh """
                            aws s3 sync build/ s3://${S3_BUCKET} --delete
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            mail to: "${RECIPIENT_EMAIL}",
                subject: "Pipeline ${env.JOB_NAME} ejecución correcta",
                body: """
                Hola,

                El pipeline '${env.JOB_NAME}' (Build #${env.BUILD_NUMBER}) ha finalizado correctamente.

                Se ha guardado una nueva versión del backup en:
                s3://${BACKUP_BUCKET}/JesusRamirez/${baseVersion}

                Detalles en:
                ${env.BUILD_URL}

                Saludos,
                Jenkins Server
                """
        }
        failure {
            mail to: "${RECIPIENT_EMAIL}",
                subject: "⚠️ Pipeline ${env.JOB_NAME} falló",
                body: """
                Hola,

                El pipeline '${env.JOB_NAME}' (Build #${env.BUILD_NUMBER}) ha fallado.

                Revisa los detalles del error en el siguiente enlace:
  
