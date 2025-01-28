stage('Restaurar último backup a bucket principal (solo main)') {
    when {
        expression {
            return env.BRANCH_NAME == 'main' // Verifica que la rama sea main
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
