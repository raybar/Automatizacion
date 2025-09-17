pipeline {
    agent any

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Clonando el repositorio...'
                git branch: 'master', url: 'https://github.com/raybar/Automatizacion.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Construyendo la imagen de Docker...'
                sh 'docker build -t automatizacion-app .'
            }
        }

        stage('Run Docker Container') {
            steps {
                echo 'Ejecutando el contenedor...'
                sh 'docker run -d --name automatizacion-app -p 80:80 automatizacion-app'
            }
        }

        stage('OWASP ZAP Dynamic Scan') {
            steps {
                script {
                    echo 'Iniciando el escaneo dinámico con OWASP ZAP...'
                    // Ejecuta un contenedor de ZAP para escanear la aplicación.
                    // Se conecta a http://host.docker.internal:8080 para acceder al contenedor de la aplicación.
                    // -v $(pwd):/zap/wrk/:rw mapea el directorio local para guardar el reporte.
                    sh 'docker run --rm -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py -t http://host.docker.internal:8080 -r zap-report.html'
                }
            }
        }

        stage('Reporting') {
            steps {
                echo 'Archivando los reportes...'
                // Archiva el reporte HTML generado por ZAP.
                archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
            }
        }
    }

    post {
        always {
            echo 'Limpiando los contenedores...'
            // Detiene y elimina ambos contenedores, la aplicación y ZAP.
            sh 'docker stop automatizacion-app || true'
            sh 'docker rm automatizacion-app || true'
        }
    }
}

