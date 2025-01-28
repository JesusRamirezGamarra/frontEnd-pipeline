pipeline {
    agent {
        docker { image 'node:16-alpine'}
    }

    stages {
        stage ('Instalar dependencias...') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analisis...') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner';
                    withSonarQubeEnv() {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('SonarQube quality gate...') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage ('Construir proyecto con archivos estaticos...') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Validar conexion AWS ...') {
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: 'us-east-1') {
                    script {
                        def buckets  = sh(returnStdout: true, script: 'aws s3 ls').trim()
                        echo "Buckets disponibles en aws: \n${buckets}"
                    }
                }
            }

        }

        stage('Subir proyecto al bucket s3 AWS ...') {
            steps {
                withAWS(credentials: 'aws-credentials-s3', region: 'us-east-1') {
                    s3Upload(bucket: 'codigo-pipeline-bucket', includePathPattern: '**/*', path: 'build/')
                                        
                }
            }

        }
    }
}