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

        stage('Validar imagen de AWS ...') {
            agent {
                docker { 
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                echo "Usando aws CLI.."
                sh 'aws --version'
            }
        }

        stage('Validar conexión AWS ...') {
            when {
                not { branch 'develop' } // Ejecutar si NO es la rama `develop`
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
                        def buckets = sh(returnStdout: true, script: 'aws s3 ls').trim()
                        echo "Buckets disponibles en AWS: \n${buckets}"
                    }
                }
            }
        }

        stage('Preparar estructura de buckets (solo main)') {
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
                        // Crear bucket backup si no existe
                        sh """
                            aws s3api head-bucket --bucket ${BACKUP_BUCKET} || aws s3 mb s3://${BACKUP_BUCKET}
                        """
                        // Crear sub-bucket "JesusRamirez" dentro del bucket backup
                        sh """
                            aws s3api put-object --bucket ${BACKUP_BUCKET} --key JesusRamirez/
                        """
                        // Crear sub-bucket con formato de fecha dentro de "JesusRamirez"
                        def timestamp = sh(
                            returnStdout: true,
                            script: "date +%Y_%m_%d_%H_%M_%S"
                        ).trim()
                        echo "Creando bucket con formato de fecha: ${timestamp} dentro de JesusRamirez..."
                        sh """
                            aws s3api put-object --bucket ${BACKUP_BUCKET} --key JesusRamirez/${timestamp}/
                        """
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