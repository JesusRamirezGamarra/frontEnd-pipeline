pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1' // Región de AWS
        S3_BUCKET = 'bucket-codigo-jesus' // Nombre del bucket principal
        BACKUP_BUCKET = 'bucket-codigo-backup' // Nombre del bucket de respaldo
    }

    stages {
        stage('Verificar rama activa y restaurar backup') {
            steps {
                script {
                    // Verificar que se está ejecutando en la rama main
                    echo "Verificando rama activa: ${env.BRANCH_NAME}"
                    if (env.BRANCH_NAME != 'main') {
                        echo "Esta ejecución no es para la rama main. Saltando restauración..."
                        return
                    }

                    // Buscar el directorio más reciente en el bucket de respaldo
                    echo "Buscando el directorio más reciente en ${BACKUP_BUCKET}/JesusRamirez/..."
                    def latestDir = sh(
                        returnStdout: true,
                        script: """
                            aws s3 ls s3://${BACKUP_BUCKET}/JesusRamirez/ | awk '{print \$2}' | sort -r | head -n 1 | tr -d '/'
                        """
                    ).trim()
                    echo "El directorio más reciente encontrado es: ${latestDir}"

                    if (latestDir) {
                        // Limpiar el contenido del bucket principal
                        echo "Eliminando contenido actual del bucket principal ${S3_BUCKET}..."
                        sh """
                            aws s3 rm s3://${S3_BUCKET}/ --recursive
                        """

                        // Restaurar contenido desde el backup
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

    post {
        success {
            echo "Pipeline ejecutado correctamente."
        }
        failure {
            echo "El pipeline falló. Revisa los errores."
        }
    }
}
