pipeline {
    agent any

    environment {
        WEBHOOK_URL = 'https://c6c6-38-25-17-72.ngrok-free.app/github-webhook/' // URL del webhook
        AWS_REGION = 'us-east-1' // Región de AWS
        S3_BUCKET = 'bucket-codigo-jesus' // Nombre del bucket S3
        RECIPIENT_EMAIL = 'luciojesusramirezgamarra@gmail.com' // Correo del destinatario
        BACKUP_BUCKET = 'bucket-codigo-backup' // Bucket de respaldo
        BACKUP_PREFIX = 'JesusRamirez/main/' // Ruta dentro del bucket backup
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
                            aws s3api head-bucket --bucket ${env.BACKUP_BUCKET} || aws s3 mb s3://${env.BACKUP_BUCKET}
                        """

                        // Obtener la última versión existente en el bucket
                        def lastVersion = sh(
                            returnStdout: true,
                            script: """
                                aws s3 ls s3://${env.BACKUP_BUCKET}/${env.BACKUP_PREFIX} | grep Version_ | awk '{print \$2}' | sort -V | tail -n1 | sed 's/\\///'
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
                            aws s3api put-object --bucket ${env.BACKUP_BUCKET} --key ${env.BACKUP_PREFIX}${newVersion}/
                        """

                        // Copiar contenido de `bucket-codigo-jesus` al nuevo backup
                        echo "Copiando contenido de ${env.S3_BUCKET} a ${env.BACKUP_BUCKET}/${env.BACKUP_PREFIX}${newVersion}/..."
                        sh """
                            aws s3 sync s3://${env.S3_BUCKET}/ s3://${env.BACKUP_BUCKET}/${env.BACKUP_PREFIX}${newVersion}/
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
                            aws s3 sync build/ s3://${env.S3_BUCKET} --delete
                        """
                    }
                }
            }
        }
    }
}
