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
                echo 'Iniciando análisis estático con el servidor local...'
                // Usamos withCredentials para acceder de forma segura al token
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    dir('dvwa') {
                        // Pasamos la URL del servidor local y el token directamente al comando
                        sh '''/usr/local/bin/sonar-scanner \
                            -Dsonar.projectKey=DVWA-Proyecto \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://sonarqube:9000 \
                            -Dsonar.login=$SONAR_TOKEN'''
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
# Asegúrate de instalar las dependencias primero
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    build-essential \
    libpq-dev \
&& docker-php-ext-install pdo pdo_mysql mysqli gd
RUN docker-php-ext-install mysqli pdo pdo_mysql
COPY . /var/www/html/
# Crea los directorios antes de intentar cambiar permisos
RUN mkdir -p /var/www/html/hackable/uploads \
&& chmod 777 /var/www/html/hackable/uploads
RUN mkdir -p /var/www/html/external/ \
&& chmod 777 /var/www/html/external/
RUN mkdir -p /var/www/html/external/phpids/0.6/lib/IDS/tmp/ \
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













