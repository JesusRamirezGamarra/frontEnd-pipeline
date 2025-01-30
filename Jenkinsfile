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
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: 'us-east-1') {
                    sh "mkdir -p build"
                    sh """
                        aws s3 sync s3://${params.BUCKET_FUENTE}/${params.CARPETA_USUARIO}/${params.CARPETA_RAMA}/${params.CARPETA_FUENTE}/ build/
                    """
                }
            }
        }

        stage('Desplegar hacia vercel') {
            when { expression { return params.DESPLEGAR_A_VERCEL } }
            steps {
                script {
                    sh "ls -la build/"
                    def buildFiles = sh(script: "ls -1 build/ | wc -l", returnStdout: true).trim()
                    if (buildFiles == "0") {
                        error("❌ Error: La carpeta build/ está vacía, no se puede desplegar en Vercel.")
                    }

                    sh """
                        npm install -g vercel
                        cd build
                        vercel deploy --prod --name front-vercel --token $VERCEL_TOKEN --yes --force
                    """
                }
            }
        }
    }
}
