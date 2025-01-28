pipeline {
    agent any

    environment {
        WEBHOOK_URL = 'https://c6c6-38-25-17-72.ngrok-free.app/github-webhook/' // URL del webhook
        AWS_REGION = 'us-east-1' // Región de AWS
        S3_BUCKET = 'bucket-codigo-jesus' // Nombre del bucket S3
        RECIPIENT_EMAIL = 'luciojesusramirezgamarra@gmail.com' 
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

        stage('Subir proyecto al bucket S3 de AWS ...') {
            when {
                not { branch 'develop' } // Ejecutar si NO es la rama `develop`
                not { branch 'QA' }      // También omitir si es la rama `QA`
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
            script {
                echo "Pipeline ejecutado con éxito."

                // Enviar correo en caso de éxito
                emailext(
                    to: "${RECIPIENT_EMAIL}",
                    subject: "Pipeline ejecutado con éxito: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                    body: """
                    <h3>Pipeline finalizado correctamente</h3>
                    <p>El pipeline <strong>${env.JOB_NAME}</strong> se ejecutó con éxito en la rama <strong>${env.BRANCH_NAME}</strong>.</p>
                    <p>Revisa los resultados en el siguiente enlace: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    mimeType: 'text/html'
                )
            }
        }
        failure {
            script {
                echo "Pipeline falló. Notificando por correo..."

                // Enviar correo en caso de fallo
                emailext(
                    to: "${RECIPIENT_EMAIL}",
                    subject: "Pipeline falló: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                    body: """
                    <h3>Pipeline falló</h3>
                    <p>El pipeline <strong>${env.JOB_NAME}</strong> falló en la rama <strong>${env.BRANCH_NAME}</strong>.</p>
                    <p>Revisa los logs en el siguiente enlace: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    mimeType: 'text/html'
                )
            }
        }
    }
}