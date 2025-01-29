pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1' // Región de AWS
        S3_BUCKET = 'bucket-codigo-jesus' // Nombre del bucket principal
        BACKUP_BUCKET = 'bucket-codigo-backup' // Nombre del bucket de respaldo
    }

    stages {
        stage('Verificar rama activa y restaurar backup') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: "${AWS_REGION}") {
                    script {
                        echo "Buscando el directorio más reciente en ${BACKUP_BUCKET}/JesusRamirez/..."
                        def latestDir = sh(
                            returnStdout: true,
                            script: """
                                aws s3 ls s3://${BACKUP_BUCKET}/JesusRamirez/ | awk '{print \$2}' | sort -r | head -n 1 | tr -d '/'
                            """
                        ).trim()
                        echo "El directorio más reciente encontrado es: ${latestDir}"

                        if (latestDir) {
                            echo "Eliminando contenido actual del bucket principal ${S3_BUCKET}..."
                            sh """
                                aws s3 rm s3://${S3_BUCKET}/ --recursive
                            """

                            echo "Restaurando contenido desde ${BACKUP_BUCKET}/JesusRamirez/${latestDir} a ${S3_BUCKET}..."
                            sh """
                                aws s3 sync s3://${BACKUP_BUCKET}/JesusRamirez/${latestDir}/ s3://${S3_BUCKET}/
                            """
                        } else {
                            echo "No se encontró ningún directorio de respaldo. Restauración omitida."
                        }
                    }
                }
            }
        }
    }
}
