pipeline {
    agent any

    stages {
        stage('Clonar Repositorio') {
            steps {
                echo 'Clonando el repositorio de la aplicación DVWA...'
                git url: 'https://github.com/raybar/Automatizacion.git', branch: 'master'
            }
        }

     stage('Análisis Estático con SonarQube') {
            steps {
                echo 'Iniciando análisis estático...'
                // Usa el nombre del servidor configurado en Jenkins
                withSonarQubeEnv('SonarQube-local') {
                    dir('dvwa') {
                        // El comando sonar-scanner ahora es reconocido
                        sh 'sonarqube-local -Dsonar.projectKey=DVWA-Proyecto'
                    }
                }
            }
        }

        stage('Crear Dockerfile') {
            steps {
                echo 'Creando el Dockerfile para DVWA...'
                dir('dvwa') {
                    sh '''
cat <<EOF > Dockerfile
FROM php:7.4-apache
RUN a2enmod rewrite
RUN docker-php-ext-install mysqli pdo pdo_mysql
COPY . /var/www/html/
RUN chmod 777 /var/www/html/hackable/uploads
RUN chmod 777 /var/www/html/external/
RUN chmod 777 /var/www/html/external/phpids/0.6/lib/IDS/tmp/
EOF
                    '''
                }
            }
        }

        stage('Construir y Desplegar Aplicación') {
            steps {
                echo 'Construyendo y ejecutando el contenedor de la aplicación DVWA...'
                sh 'docker stop dvwa-app || true'
                sh 'docker rm dvwa-app || true'

                dir('dvwa') {
                    sh 'docker build -t dvwa-image .'
                }

                sh 'docker run -d --name dvwa-app -p 8080:80 dvwa-image'
                sleep 30
            }
        }

        stage('Análisis Dinámico con OWASP ZAP') {
            steps {
                script {
                    echo 'Iniciando el escaneo dinámico con OWASP ZAP...'
                    sh 'docker run --rm -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py -t http://host.docker.internal:8080 -r zap-report.html'
                }
            }
        }

        stage('Reportes') {
            steps {
                echo 'Archivando los reportes de análisis...'
                archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
            }
        }
    }

    post {
        always {
            echo 'Limpiando el contenedor y los archivos temporales...'
            sh 'docker stop dvwa-app || true'
            sh 'docker rm dvwa-app || true'
            sh 'rm -f ./dvwa/Dockerfile || true'
        }
    }
}









