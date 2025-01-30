pipeline {
    agent any

    environment {
        VERCEL_TOKEN = credentials('VERCEL_TOKEN')
    }

    parameters {
        string(name: 'BUCKET_FUENTE', defaultValue: 'bucket-codigo-backup', description: 'Nombre del bucket de origen..')
        string(name: 'BUCKET_TARGET', defaultValue: 'bucket-codigo-jesus', description: 'Nombre del bucket objetivo..')
        string(name: 'CARPETA_USUARIO', defaultValue: 'JesusRamirez', description: 'Nombre de la carpeta del usuario..')
        string(name: 'CARPETA_FUENTE', defaultValue: 'VERSION_1.0', description: 'Nombre de la carpeta del bucket origen..')
        string(name: 'CARPETA_RAMA', defaultValue: 'main', description: 'Nombre de la carpeta de la rama del proyecto')
        booleanParam(name: 'DESPLEGAR_A_VERCEL', defaultValue: false, description: 'Deseas desplegar en Vercel?')
    }

    stages {
        stage('Validar parámetros...') {
            steps {
                script {
                    echo "BUCKET DE ORIGEN: ${params.BUCKET_FUENTE}"
                    echo "BUCKET DE DESTINO: ${params.BUCKET_TARGET}"
                    echo "CARPETA DE USUARIO: ${params.CARPETA_USUARIO}"
                    echo "CARPETA DE ORIGEN: ${params.CARPETA_FUENTE}"
                    echo "CARPETA RAMA: ${params.CARPETA_RAMA}"
                    echo "FLAG PARA DESPLEGAR EN VERCEL: ${params.DESPLEGAR_A_VERCEL}"
                }
            }
        }

        stage('Descargar S3 hacia carpeta build') {
            when { expression { return params.DESPLEGAR_A_VERCEL } }
            
            agent {
                docker {
                    image 'amazon/aws-cli:2.23.7'
                    args '--entrypoint ""'
                }
            }

            steps {
                withAWS(credentials: 'aws-credentials-s3', region: 'us-east-1') {
                    script {
                        sh "mkdir -p build"

                        echo "Verificando si la carpeta en S3 existe..."
                        def carpetaExiste = sh(script: """
                            aws s3 ls s3://${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.CARPETA_RAMA}/ | grep ${params.CARPETA_FUENTE} || echo 'NO_EXISTE'
                        """, returnStdout: true).trim()

                        if (carpetaExiste == "NO_EXISTE") {
                            error("❌ ERROR: La carpeta S3 ${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.CARPETA_RAMA}/${params.CARPETA_FUENTE} no existe.")
                        }

                        echo "Descargando archivos desde S3..."
                        sh """
                            aws s3 sync s3://${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.CARPETA_RAMA}/${params.CARPETA_FUENTE}/ build/
                        """

                        echo "Listando archivos descargados..."
                        sh "ls -la build/"
                    }
                }
            }
        }

        stage('Desplegar hacia vercel') {
            when { expression { return params.DESPLEGAR_A_VERCEL } }

            agent {
                docker { image 'node:18-alpine' }
            }

            steps {
                script {
                    echo "Listando archivos antes del deploy..."
                    sh "ls -la build/"

                    def buildFiles = sh(script: "ls -1 build/ | wc -l", returnStdout: true).trim()
                    if (buildFiles == "0") {
                        error("❌ Error: La carpeta build/ está vacía, no se puede desplegar en Vercel.")
                    }

                    echo "Instalando dependencias..."
                    sh """
                        cp package.json build/
                        cd build
                        npm install --omit=dev
                    """

                    echo "Desplegando con Vercel..."
                    sh """
                        npm install -g vercel
                        cd build
                        vercel deploy --prod --token $VERCEL_TOKEN --yes --force
                    """
                }
            }
        }
    }

    post {
        success {
            mail to: 'luciojesusramirezgamarra@gmail.com',
                subject: "✅ Pipeline ${env.JOB_NAME} ejecutado correctamente",
                body: """
                Hola,

                El pipeline '${env.JOB_NAME}' (Build #${env.BUILD_NUMBER}) ha finalizado correctamente.

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
