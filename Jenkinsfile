pipeline {
    agent any

    environment {
        WEBHOOK_URL = 'https://c6c6-38-25-17-72.ngrok-free.app/github-webhook/' // URL del webhook
        AWS_REGION = 'us-east-1' // Región de AWS
        S3_BUCKET = 'bucket-codigo-jesus' // Nombre del bucket S3
        RECIPIENT_EMAIL = 'luciojesusramirezgamarra@gmail.com' // Correo del destinatario
        BACKUP_BUCKET = 'bucket-codigo-backup' // Bucket para el respaldo
        BACKUP_PREFIX = 'JesusRamirez/Vercel/' // Nueva estructura para los backups
    }

    stages {
        stage ('Instalar dependencias...') {
            agent {
                docker { image 'node:16-alpine' }
            }
            steps {
                sh 'npm install'
            }
        }

        stage ('Construir proyecto con archivos estáticos...') {
            agent {
                docker { image 'node:16-alpine' }
            }
            steps {
                sh 'npm run build'
            }
        }

        stage('Preparar estructura de buckets y realizar backup (solo main)') {
            when {
                branch 'main' // Solo ejecutar en la rama `main`
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
                        // Crear el bucket de backup si no existe
                        sh """
                            aws s3api head-bucket --bucket ${BACKUP_BUCKET} || aws s3 mb s3://${BACKUP_BUCKET}
                        """

                        // Obtener la última versión existente en el bucket
                        def lastVersion = sh(
                            returnStdout: true,
                            script: """
                                aws s3 ls s3://${BACKUP_BUCKET}/${BACKUP_PREFIX} | grep Version_ | awk '{print $2}' | sort -V | tail -n1 | sed 's/\\///'
                            """
                        ).trim()

                        // Si no hay versiones previas, comenzar desde 1.0
                        def newVersion
                        if (lastVersion) {
                            def versionNumber = lastVersion.replace("Version_", "").toFloat() + 0.1
                            newVersion = sprintf("Version_%.1f", versionNumber)
                        } else {
                            newVersion = "Version_1.0"
                        }

                        echo "Nueva versión de backup: ${newVersion}"

                        // Crear la nueva carpeta en el bucket de backup
                        sh """
                            aws s3api put-object --bucket ${BACKUP_BUCKET} --key ${BACKUP_PREFIX}${newVersion}/
                        """

                        // Copiar contenido de `bucket-codigo-jesus` al nuevo backup
                        echo "Copiando contenido de ${S3_BUCKET} a ${BACKUP_BUCKET}/${BACKUP_PREFIX}${newVersion}/..."
                        sh """
                            aws s3 sync s3://${S3_BUCKET}/ s3://${BACKUP_BUCKET}/${BACKUP_PREFIX}${newVersion}/
                        """
                    }
                }
            }
        }        

        stage('Subir proyecto al bucket S3 de AWS ...') {
            when {
                not { branch 'develop' } // No ejecutar en la rama `develop`
                not { branch 'QA' }      // No ejecutar en la rama `QA`
                branch 'main'            // Ejecutar solo en la rama `main`
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
    }

    post {
        success {
            mail to: 'luciojesusramirezgamarra@gmail.com',
                subject: "✅ Pipeline ${env.JOB_NAME} ejecucion correcta",
                body: """
                Hola,

                El pipeline '${env.JOB_NAME}' (Build #${env.BUILD_NUMBER}) ha finalizado de manera correcta.

                Detalles del backup:
                - Versión creada: ${newVersion}
                - Ubicación en S3: s3://${BACKUP_BUCKET}/${BACKUP_PREFIX}${newVersion}/

                Puedes revisar más detalles en:
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
