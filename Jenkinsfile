pipeline {
    agent any

    parameters {
        string(name: 'BUCKET_FUENTE', defaultValue: 'bucket-codigo-backup', description: 'Nombre del bucket de origen..')
        string(name: 'BUCKET_TARGET', defaultValue: 'bucket-codigo-jesus', description: 'Nombre del bucket objetivo..')
        string(name: 'CARPETA_USUARIO', defaultValue: 'JesusRamirez', description: 'Nombre de la carpeta del usuario..')
        string(name: 'CARPETA_FUENTE', defaultValue: 'frontend_fuente_v1', description: 'Nombre de la carpeta del bucket origen..')
        string(name: 'BRANCH_NAME', defaultValue: 'QA', description: 'Nombre de la Rama name /  bucket destino ..')
    }


    stages {
        stage('Validar parametros...') {
            agent any
            steps {
                script {
                    echo "BUCKET DE ORIGEN: ${params.BUCKET_FUENTE}"
                    echo "BUCKET DE DESTINO: ${params.BUCKET_TARGET}"
                    echo "CARPETA DE USUARIO: ${params.CARPETA_USUARIO}"
                    echo "CARPETA DE ORIGEN: ${params.CARPETA_FUENTE}"
                    echo "BRANCH_NAME: ${params.BRANCH_NAME}"
                }
            }
        }

        stage('Listar archivos en el bucket fuente') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: 'us-east-1') {
                    script {
                        echo "Listando archivos en el bucket fuente antes de copiar..."
                        sh """
                            aws s3 ls s3://${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.BRANCH_NAME}/${params.CARPETA_FUENTE}/ --recursive
                        """
                    }
                }
            }
        }

        stage('Mover archivos entre buckets s3 AWS ...') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: 'us-east-1') {
                    script {
                        echo "Limpiando bucket objetivo..."

                        sh """
                            aws s3 rm s3://${params.BUCKET_TARGET}/ --recursive
                        """

                        echo "Sincronizando archivos entre buckets s3..."
                        sh """
                            aws s3 sync s3://${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.CARPETA_FUENTE}/ s3://${params.BUCKET_TARGET}/ --delete
                        """
                    }                   
                }
            }
        }
    }
}
