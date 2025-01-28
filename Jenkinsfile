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

        stage('Restaurar último backup a bucket principal (solo main)') {
            when {
                expression {
                    return env.BRANCH_NAME == 'main' // Ejecutar solo si la rama es main
                }
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
                        // Buscar la fecha máxima existente en el bucket de respaldo
                        echo "Buscando el directorio más reciente en ${BACKUP_BUCKET}/JesusRamirez/..."
                        def latestDir = sh(
                            returnStdout: true,
                            script: """
                                aws s3 ls s3://${BACKUP_BUCKET}/JesusRamirez/ | awk '{print \$2}' | sort -r | head -n 1 | tr -d '/'
                            """
                        ).trim()
                        echo "El directorio más reciente encontrado es: ${latestDir}"

                        if (latestDir) {
                            // Verificar si existe contenido en el directorio encontrado
                            echo "Verificando contenido en ${BACKUP_BUCKET}/JesusRamirez/${latestDir}/..."
                            def dirContent = sh(
                                returnStdout: true,
                                script: """
                                    aws s3 ls s3://${BACKUP_BUCKET}/JesusRamirez/${latestDir}/
                                """
                            ).trim()
                            if (dirContent) {
                                // Limpiar el contenido del bucket principal
                                echo "Eliminando contenido actual del bucket principal ${S3_BUCKET}..."
                                sh """
                                    aws s3 rm s3://${S3_BUCKET}/ --recursive
                                    aws s3 ls s3://${S3_BUCKET}/
                                """

                                // Copiar el contenido del último backup al bucket principal
                                echo "Restaurando contenido desde ${BACKUP_BUCKET}/JesusRamirez/${latestDir} a ${S3_BUCKET}..."
                                sh """
                                    aws s3 sync s3://${BACKUP_BUCKET}/JesusRamirez/${latestDir}/ s3://${S3_BUCKET}/
                                """
                            } else {
                                echo "El directorio ${latestDir} está vacío. Restauración omitida."
                            }
                        } else {
                            echo "No se encontró ningún directorio de respaldo. Restauración omitida."
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            mail to: 'luciojesusramirezgamarra@gmail.com',
                subject: "Pipeline ${env.JOB_NAME} ejecucion correcta",
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
