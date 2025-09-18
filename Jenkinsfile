pipeline {
    agent any
    stages {
        stage('Clonar Repositorio') {
            steps {
                echo 'Clonando el repositorio de la aplicación DVWA...'
                git url: 'https://github.com/raybar/Automatizacion.git', branch: 'master'
                sh 'ls -R'
            }
        }
        stage('Análisis Estático con SonarQube') {
            steps {
                echo 'Iniciando análisis estático con el servidor local...'
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    dir('dvwa') {
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
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    build-essential \
    libpq-dev \
&& docker-php-ext-install pdo pdo_mysql mysqli gd
RUN docker-php-ext-install mysqli pdo pdo_mysql
COPY . /var/www/html/
RUN mkdir -p /var/www/html/hackable/uploads \
&& chmod 777 /var/www/html/hackable/uploads
RUN mkdir -p /var/www/html/external/ \
&& chmod 777 /var/www/html/external/
RUN mkdir -p /var/www/html/external/phpids/0.6/lib/IDS/tmp/ \
&& chmod 777 /var/www/html/external/phpids/0.6/lib/IDS/tmp/
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
                sh 'docker run -d --name dvwa-app dvwa-image'
                sleep 30
            }
        }
        stage('Análisis Dinámico con OWASP ZAP') {
            steps {
                script {
                    echo 'Iniciando el escaneo dinámico con OWASP ZAP...'
                    sh 'ls -R $(pwd)'
                    sh 'docker run --rm -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py -t http://dvwa-app:80 -r /zap/wrk/zap-report.html'
                }
            }
        }
        stage('Reportes') {
            steps {
                echo 'Archivando los reportes de análisis...'
                sh 'ls -R'
                archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
            }
        }
    }
    // Bloque 'post' correcto: se encuentra al mismo nivel que 'stages'
    post {
        always {
            echo 'Limpiando el contenedor y los archivos temporales...'
            sh 'docker stop dvwa-app || true'
            sh 'docker rm dvwa-app || true'
            sh 'rm -f ./dvwa/Dockerfile || true'
        }
    }
}





