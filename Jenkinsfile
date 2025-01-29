pipeline {
    agent any

    parameters {
        string(name: 'BUCKET_FUENTE', defaultValue: 'bucket-codigo-backup', description: 'Nombre del bucket de origen.')
        string(name: 'BUCKET_TARGET', defaultValue: 'bucket-codigo-front', description: 'Nombre del bucket de destino.')
        string(name: 'CARPETA_USUARIO', defaultValue: 'jesusramirez', description: 'Carpeta del usuario.')
        string(name: 'CARPETA_FUENTE', defaultValue: 'frontend-fuente-v1', description: 'Carpeta de origen dentro del bucket.')
    }

    environment {
        AWS_REGION = 'us-east-1' // Región de AWS
        RECIPIENT_EMAIL = 'luciojesusramirezgamarra@gmail.com' // Email de notificación
    }

    stages {
        stage('Validar parámetros') {
            steps {
                script {
                    echo "Parámetros recibidos:"
                    echo "BUCKET DE ORIGEN: ${params.BUCKET_FUENTE}"
                    echo "BUCKET DE DESTINO: ${params.BUCKET_TARGET}"
                    echo "CARPETA DE USUARIO: ${params.CARPETA_USUARIO}"
                    echo "CARPETA DE ORIGEN: ${params.CARPETA_FUENTE}"
                }
            }
        }

        stage('Mover archivos entre buckets S3') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: "${AWS_REGION}") {
                    script {
                        echo "Iniciando movimiento de archivos desde '${params.BUCKET_FUENTE}' a '${params.BUCKET_TARGET}'..."
                        sh """
                            aws s3 mv s3://${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.CARPETA_FUENTE}/ s3://${params.BUCKET_TARGET}/ --recursive
                        """
                        echo "Movimiento completado."
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

                Los archivos fueron movidos de:
                - Origen: s3://${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.CARPETA_FUENTE}/
                - Destino: s3://${params.BUCKET_TARGET}/

                Detalles del pipeline:
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

                Verifica los logs en el siguiente enlace:
                ${env.BUILD_URL}

                Saludos,
                Jenkins Server
                """
        }
    }
}
