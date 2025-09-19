pipeline {
    agent any
    
    environment {
        // Variables de entorno para el pipeline
        DVWA_IMAGE = 'dvwa:latest'
        DVWA_CONTAINER = 'dvwa-app'
        MYSQL_CONTAINER = 'dvwa-mysql'
        DVWA_NETWORK = 'dvwa-network'
        ZAP_NETWORK = 'zap-net'
        DVWA_PORT = '80'
        BUILD_TIMESTAMP = "${new Date().format('yyyyMMdd-HHmmss')}"
    }
    
    options {
        // Configuraciones del pipeline
        timeout(time: 30, unit: 'MINUTES')
        retry(2)
        skipDefaultCheckout(false)
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    stages {
        stage('Preparación del Entorno') {
            steps {
                echo "🚀 Iniciando pipeline DevSecOps para DVWA - Build ${BUILD_TIMESTAMP}"
                script {
                    // Verificar que Docker esté disponible
                    sh 'docker --version'
                    sh 'docker compose version'
                    
                    // Verificar herramientas necesarias
                    sh '''
                        # Verificar si las herramientas están disponibles
                        which curl || echo "⚠️ curl no está disponible"
                        which git || echo "⚠️ git no está disponible"
                    '''
                    
                    // Crear redes si no existen
                    sh '''
                        docker network create ${DVWA_NETWORK} || echo "Red ${DVWA_NETWORK} ya existe"
                        docker network create ${ZAP_NETWORK} || echo "Red ${ZAP_NETWORK} ya existe"
                    '''
                }
            }
        }
        
        stage('Clonar Repositorio') {
            steps {
                echo '📥 Clonando el repositorio de la aplicación DVWA...'
                git url: 'https://github.com/raybar/Automatizacion.git', branch: 'master'
                
                // Verificar estructura del proyecto
                sh '''
                    echo "📁 Estructura del proyecto:"
                    find . -maxdepth 2 -type f -name "*.php" | head -10
                    ls -la
                '''
            }
        }
        
        stage('Análisis Estático con SonarQube') {
            steps {
                echo '🔍 Iniciando análisis estático con SonarQube...'
                script {
                    try {
                        // Usar imagen Docker de SonarQube Scanner con conectividad al host
                        // Usamos withCredentials para acceder de forma segura al token
                        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')])
                        sh '''
                            # Verificar conectividad con SonarQube
                            if curl -s --connect-timeout 5 http://sonarqube:9000/api/system/status > /dev/null; then
                                echo "✅ SonarQube está disponible, ejecutando análisis..."
                                docker run --rm \
                                    --add-host=host.docker.internal:host-gateway \
                                    -v $(pwd):/usr/src \
                                    -e SONAR_HOST_URL=http://host.docker.internal:9000 \
                                    -e SONAR_SCANNER_OPTS="-Dsonar.projectKey=DVWA-Proyecto-${BUILD_TIMESTAMP}" \
                                    sonarsource/sonar-scanner-cli:latest \
                                    /usr/local/bin/sonar-scanner \
                                        -Dsonar.projectName="DVWA Security Analysis" \
                                        -Dsonar.projectVersion=${BUILD_NUMBER} \
                                        -Dsonar.login=$SONAR_TOKEN \
                                        -Dsonar.sources=./dvwa \
                                        -Dsonar.exclusions="**/*.jpg,**/*.png,**/*.gif,**/*.pdf" \
                                        -Dsonar.php.coverage.reportPaths=coverage.xml
                            else
                                echo "⚠️ SonarQube no está disponible, omitiendo análisis estático"
                                echo "Para habilitar SonarQube, asegúrese de que esté ejecutándose en localhost:9000"
                            fi
                        '''
                        
                        // Análisis estático básico alternativo
                        echo "🔍 Ejecutando análisis estático básico..."
                        sh '''
                            # Crear directorio para reportes de análisis
                            mkdir -p static-analysis
                            
                            # Análisis básico de archivos PHP
                            echo "📋 Análisis de archivos PHP:"
                            find ./dvwa -name "*.php" -type f | head -20 > static-analysis/php-files.txt
                            echo "Archivos PHP encontrados: $(wc -l < static-analysis/php-files.txt)"
                            
                            # Buscar patrones de seguridad básicos
                            echo "🔒 Buscando patrones de seguridad potencialmente problemáticos:"
                            
                            # Buscar uso de funciones peligrosas
                            grep -r "eval\\|exec\\|system\\|shell_exec\\|passthru" ./dvwa --include="*.php" > static-analysis/dangerous-functions.txt 2>/dev/null || echo "No se encontraron funciones peligrosas"
                            
                            # Buscar SQL directo (posibles inyecciones)
                            grep -r "SELECT\\|INSERT\\|UPDATE\\|DELETE" ./dvwa --include="*.php" | grep -v "//\\|#\\|\\*" > static-analysis/sql-queries.txt 2>/dev/null || echo "No se encontraron consultas SQL directas"
                            
                            # Buscar inclusiones de archivos dinámicas
                            grep -r "include\\|require" ./dvwa --include="*.php" | grep "\\$" > static-analysis/dynamic-includes.txt 2>/dev/null || echo "No se encontraron inclusiones dinámicas"
                            
                            # Generar reporte básico
                            cat > static-analysis/basic-security-report.txt << EOF
=== REPORTE DE ANÁLISIS ESTÁTICO BÁSICO ===
Timestamp: $(date)
Proyecto: DVWA Security Analysis

=== ESTADÍSTICAS ===
Archivos PHP analizados: $(find ./dvwa -name "*.php" -type f | wc -l)
Funciones peligrosas encontradas: $(wc -l < static-analysis/dangerous-functions.txt 2>/dev/null || echo "0")
Consultas SQL directas: $(wc -l < static-analysis/sql-queries.txt 2>/dev/null || echo "0")
Inclusiones dinámicas: $(wc -l < static-analysis/dynamic-includes.txt 2>/dev/null || echo "0")

=== RECOMENDACIONES ===
- Revisar el uso de funciones peligrosas como eval(), exec(), system()
- Validar todas las consultas SQL para prevenir inyecciones
- Sanitizar todas las inclusiones dinámicas de archivos
- Implementar validación de entrada en todos los formularios
EOF
                            
                            echo "✅ Análisis estático básico completado"
                            ls -la static-analysis/
                        '''
                        
                        // Esperar a que SonarQube procese los resultados si está disponible
                        sleep(time: 5, unit: 'SECONDS')
                        echo '✅ Análisis estático completado'
                        
                    } catch (Exception e) {
                        echo "⚠️ Error en análisis estático: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Crear y Optimizar Dockerfile') {
            steps {
                echo '🐳 Creando Dockerfile optimizado para DVWA...'
                script {
                    // Crear Dockerfile mejorado con mejores prácticas de seguridad
                    writeFile file: 'Dockerfile', text: '''
# Dockerfile optimizado para DVWA con mejores prácticas de seguridad
FROM php:8.1-apache

# Metadatos
LABEL maintainer="DevSecOps Team"
LABEL version="1.0"
LABEL description="DVWA - Damn Vulnerable Web Application para testing de seguridad"

# Variables de entorno
ENV DEBIAN_FRONTEND=noninteractive
ENV APACHE_DOCUMENT_ROOT=/var/www/html
ENV MYSQL_HOST=mysql
ENV MYSQL_DATABASE=dvwa
ENV MYSQL_USER=root
ENV MYSQL_PASSWORD=dvwa

# Actualizar sistema y instalar dependencias necesarias
RUN apt-get update && apt-get install -y --no-install-recommends \\
    libpng-dev \\
    libjpeg62-turbo-dev \\
    libfreetype6-dev \\
    libzip-dev \\
    libonig-dev \\
    default-mysql-client \\
    curl \\
    wget \\
    && rm -rf /var/lib/apt/lists/*

# Instalar extensiones PHP necesarias
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \\
    && docker-php-ext-install -j$(nproc) \\
        pdo \\
        pdo_mysql \\
        mysqli \\
        gd \\
        zip \\
        mbstring

# Habilitar módulos de Apache
RUN a2enmod rewrite headers

# Crear usuario no privilegiado para la aplicación
RUN groupadd -r dvwa && useradd -r -g dvwa dvwa

# Copiar archivos de la aplicación
COPY . ${APACHE_DOCUMENT_ROOT}/

# Configurar DVWA
RUN if [ -f ${APACHE_DOCUMENT_ROOT}/config/config.inc.php.dist ]; then \\
        cp ${APACHE_DOCUMENT_ROOT}/config/config.inc.php.dist ${APACHE_DOCUMENT_ROOT}/config/config.inc.php; \\
    fi

# Configurar permisos de seguridad
RUN chown -R www-data:www-data ${APACHE_DOCUMENT_ROOT} \\
    && find ${APACHE_DOCUMENT_ROOT} -type d -exec chmod 755 {} \\; \\
    && find ${APACHE_DOCUMENT_ROOT} -type f -exec chmod 644 {} \\;

# Crear directorios necesarios con permisos específicos
RUN mkdir -p ${APACHE_DOCUMENT_ROOT}/hackable/uploads \\
    && chmod 775 ${APACHE_DOCUMENT_ROOT}/hackable/uploads \\
    && mkdir -p ${APACHE_DOCUMENT_ROOT}/external/phpids/0.6/lib/IDS/tmp \\
    && chmod 775 ${APACHE_DOCUMENT_ROOT}/external/phpids/0.6/lib/IDS/tmp

# Configuración de seguridad de Apache
RUN echo "ServerTokens Prod" >> /etc/apache2/apache2.conf \\
    && echo "ServerSignature Off" >> /etc/apache2/apache2.conf

# Script de inicio para verificar conectividad con MySQL
COPY <<EOF /usr/local/bin/start-dvwa.sh
#!/bin/bash
set -e

echo "🔄 Esperando a que MySQL esté disponible..."
until mysqladmin ping -h"\$MYSQL_HOST" -u"\$MYSQL_USER" -p"\$MYSQL_PASSWORD" --silent; do
    echo "⏳ MySQL no está listo, esperando..."
    sleep 2
done

echo "✅ MySQL está disponible, iniciando Apache..."
exec apache2-foreground
EOF

RUN chmod +x /usr/local/bin/start-dvwa.sh

# Exponer puerto
EXPOSE 80

# Verificación de salud
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \\
    CMD curl -f http://localhost/ || exit 1

# Cambiar a usuario no privilegiado para ejecución
USER www-data

# Comando de inicio
CMD ["/usr/local/bin/start-dvwa.sh"]
'''
                }
            }
        }
        
        stage('Análisis de Seguridad del Dockerfile') {
            steps {
                echo '🔒 Analizando seguridad del Dockerfile...'
                script {
                    try {
                        // Análisis básico de seguridad del Dockerfile
                        sh '''
                            echo "📋 Verificaciones de seguridad del Dockerfile:"
                            
                            # Verificar que no se ejecute como root
                            if grep -q "USER root" Dockerfile && ! grep -q "USER www-data" Dockerfile; then
                                echo "⚠️ ADVERTENCIA: El contenedor podría ejecutarse como root"
                            else
                                echo "✅ Usuario no privilegiado configurado"
                            fi
                            
                            # Verificar HEALTHCHECK
                            if grep -q "HEALTHCHECK" Dockerfile; then
                                echo "✅ Healthcheck configurado"
                            else
                                echo "⚠️ ADVERTENCIA: No se encontró HEALTHCHECK"
                            fi
                            
                            # Verificar limpieza de cache
                            if grep -q "rm -rf /var/lib/apt/lists" Dockerfile; then
                                echo "✅ Limpieza de cache APT configurada"
                            else
                                echo "⚠️ ADVERTENCIA: Cache APT no limpiado"
                            fi
                        '''
                    } catch (Exception e) {
                        echo "⚠️ Error en análisis de Dockerfile: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Construir Imagen Docker') {
            steps {
                echo '🔨 Construyendo imagen Docker de DVWA...'
                script {
                    try {
                        sh '''
                            # Construir imagen con etiquetas múltiples
                            docker build \\
                                --tag ${DVWA_IMAGE} \\
                                --tag dvwa:${BUILD_TIMESTAMP} \\
                                --label "build.number=${BUILD_NUMBER}" \\
                                --label "build.timestamp=${BUILD_TIMESTAMP}" \\
                                --no-cache \\
                                .
                            
                            # Verificar que la imagen se creó correctamente
                            docker images | grep dvwa
                            
                            # Análisis básico de la imagen (sin dependencia de jq)
                            echo "📊 Información de la imagen:"
                            echo "🔍 Puertos expuestos:"
                            docker inspect ${DVWA_IMAGE} --format='{{range $port, $config := .Config.ExposedPorts}}{{$port}} {{end}}'
                            echo "👤 Usuario configurado:"
                            docker inspect ${DVWA_IMAGE} --format='{{.Config.User}}'
                            echo "🏥 Healthcheck configurado:"
                            docker inspect ${DVWA_IMAGE} --format='{{if .Config.Healthcheck}}✅ Sí - Intervalo: {{.Config.Healthcheck.Interval}}, Timeout: {{.Config.Healthcheck.Timeout}}{{else}}❌ No configurado{{end}}'
                            echo "📏 Tamaño de la imagen:"
                            docker images ${DVWA_IMAGE} --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
                        '''
                    } catch (Exception e) {
                        error "❌ Error al construir la imagen Docker: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Preparar Base de Datos') {
            steps {
                echo '🗄️ Preparando base de datos MySQL...'
                script {
                    try {
                        sh '''
                            # Detener y limpiar contenedor MySQL existente
                            docker stop ${MYSQL_CONTAINER} || true
                            docker rm ${MYSQL_CONTAINER} || true
                            
                            # Iniciar MySQL
                            docker run -d \\
                                --name ${MYSQL_CONTAINER} \\
                                --network ${DVWA_NETWORK} \\
                                --network ${ZAP_NETWORK} \\
                                -e MYSQL_ROOT_PASSWORD=dvwa \\
                                -e MYSQL_DATABASE=dvwa \\
                                -e MYSQL_USER=dvwa \\
                                -e MYSQL_PASSWORD=dvwa \\
                                --restart unless-stopped \\
                                mysql:8.0 \\
                                --default-authentication-plugin=mysql_native_password
                            
                            # Esperar a que MySQL esté listo
                            echo "⏳ Esperando a que MySQL esté disponible..."
                            timeout 60 bash -c 'until docker exec ${MYSQL_CONTAINER} mysqladmin ping -h localhost --silent; do sleep 2; done'
                            echo "✅ MySQL está listo"
                        '''
                    } catch (Exception e) {
                        error "❌ Error al preparar la base de datos: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Desplegar Aplicación DVWA') {
            steps {
                echo '🚀 Desplegando aplicación DVWA...'
                script {
                    try {
                        sh '''
                            # Detener y limpiar contenedor DVWA existente
                            docker stop ${DVWA_CONTAINER} || true
                            docker rm ${DVWA_CONTAINER} || true
                            
                            # Desplegar DVWA
                            docker run -d \\
                                --name ${DVWA_CONTAINER} \\
                                --network ${DVWA_NETWORK} \\
                                --network ${ZAP_NETWORK} \\
                                -p ${DVWA_PORT}:80 \\
                                -e MYSQL_HOST=${MYSQL_CONTAINER} \\
                                -e MYSQL_DATABASE=dvwa \\
                                -e MYSQL_USER=root \\
                                -e MYSQL_PASSWORD=dvwa \\
                                --restart unless-stopped \\
                                ${DVWA_IMAGE}
                            
                            echo "✅ Contenedor DVWA desplegado"
                        '''
                    } catch (Exception e) {
                        error "❌ Error al desplegar DVWA: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Verificaciones de Salud') {
            steps {
                echo '🏥 Ejecutando verificaciones de salud...'
                script {
                    try {
                        sh '''
                            # Hacer el script de diagnóstico ejecutable
                            chmod +x diagnose-containers.sh
                            
                            # Verificar permisos de Docker primero
                            echo "🔐 Verificando acceso a Docker..."
                            if ! docker version >/dev/null 2>&1; then
                                echo "❌ ERROR: No se puede acceder al daemon de Docker"
                                echo "🔧 Intentando soluciones alternativas..."
                                
                                # Verificar si Docker está ejecutándose
                                if pgrep dockerd >/dev/null 2>&1; then
                                    echo "✅ Docker daemon está ejecutándose"
                                    echo "⚠️ Problema de permisos detectado"
                                    
                                    # Intentar con sudo si está disponible
                                    if command -v sudo >/dev/null 2>&1; then
                                        echo "🔧 Intentando con sudo..."
                                        sudo docker version || echo "❌ Sudo también falló"
                                    fi
                                else
                                    echo "❌ Docker daemon no está ejecutándose"
                                    echo "🔧 Intentando iniciar Docker..."
                                    sudo systemctl start docker 2>/dev/null || echo "❌ No se pudo iniciar Docker"
                                fi
                                
                                echo "⚠️ Continuando con verificaciones limitadas..."
                            else
                                echo "✅ Acceso a Docker verificado"
                            fi
                            
                            # Ejecutar diagnóstico inicial (ahora maneja errores mejor)
                            echo "🔍 Ejecutando diagnóstico inicial de contenedores..."
                            if ./diagnose-containers.sh; then
                                echo "✅ Diagnóstico inicial completado exitosamente"
                            else
                                echo "⚠️ Diagnóstico inicial completado con advertencias"
                                echo "🔍 Continuando con verificaciones alternativas..."
                            fi
                            
                            # Verificar que los contenedores estén ejecutándose
                            echo "📊 Verificando estado de contenedores..."
                            if ! docker ps --filter "name=${MYSQL_CONTAINER}" --format "{{.Names}}" | grep -q "${MYSQL_CONTAINER}"; then
                                echo "❌ Error: Contenedor MySQL no está ejecutándose"
                                docker logs --tail 20 ${MYSQL_CONTAINER} || echo "No se pudieron obtener logs de MySQL"
                                exit 1
                            fi
                            
                            if ! docker ps --filter "name=${DVWA_CONTAINER}" --format "{{.Names}}" | grep -q "${DVWA_CONTAINER}"; then
                                echo "❌ Error: Contenedor DVWA no está ejecutándose"
                                docker logs --tail 20 ${DVWA_CONTAINER} || echo "No se pudieron obtener logs de DVWA"
                                exit 1
                            fi
                            
                            echo "✅ Ambos contenedores están ejecutándose"
                            
                            # Esperar a que la aplicación esté lista con verificaciones incrementales
                            echo "⏳ Esperando a que DVWA esté disponible..."
                            
                            # Verificación en etapas con timeouts incrementales
                            for attempt in 1 2 3 4 5; do
                                echo "🔄 Intento $attempt/5 - Verificando disponibilidad..."
                                
                                # Verificar puerto primero
                                if ! docker port ${DVWA_CONTAINER} 80 >/dev/null 2>&1; then
                                    echo "⚠️ Puerto 80 no está expuesto en el contenedor"
                                    sleep 10
                                    continue
                                fi
                                
                                # Verificar conectividad HTTP básica
                                if timeout 30 bash -c 'until curl -s http://localhost:${DVWA_PORT}/ >/dev/null 2>&1; do sleep 2; done'; then
                                    echo "✅ Conectividad HTTP establecida en intento $attempt"
                                    break
                                else
                                    echo "⚠️ Intento $attempt falló, esperando antes del siguiente..."
                                    if [ $attempt -eq 5 ]; then
                                        echo "❌ Error: DVWA no responde después de 5 intentos"
                                        echo "📋 Logs del contenedor DVWA:"
                                        docker logs --tail 30 ${DVWA_CONTAINER}
                                        echo "📋 Logs del contenedor MySQL:"
                                        docker logs --tail 20 ${MYSQL_CONTAINER}
                                        exit 1
                                    fi
                                    sleep 15
                                fi
                            done
                            
                            # Verificaciones de salud detalladas
                            echo "🔍 Ejecutando verificaciones de salud detalladas:"
                            
                            # Verificar respuesta HTTP
                            echo "🌐 Verificando respuesta HTTP..."
                            HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${DVWA_PORT}/ || echo "000")
                            if [ "$HTTP_STATUS" = "200" ]; then
                                echo "✅ Aplicación responde correctamente (HTTP $HTTP_STATUS)"
                            else
                                echo "❌ Error: Aplicación no responde correctamente (HTTP $HTTP_STATUS)"
                                echo "🔍 Intentando obtener más información..."
                                curl -v http://localhost:${DVWA_PORT}/ || echo "Curl falló completamente"
                                exit 1
                            fi
                            
                            # Verificar contenido de la página
                            echo "📄 Verificando contenido de la página..."
                            CONTENT_CHECK=$(curl -s http://localhost:${DVWA_PORT}/ | grep -c "DVWA" || echo "0")
                            if [ "$CONTENT_CHECK" -gt "0" ]; then
                                echo "✅ Contenido DVWA detectado correctamente ($CONTENT_CHECK coincidencias)"
                            else
                                echo "❌ Error: Contenido DVWA no encontrado"
                                echo "📄 Primeras líneas de la respuesta:"
                                curl -s http://localhost:${DVWA_PORT}/ | head -10
                                exit 1
                            fi
                            
                            # Verificar funcionalidad básica de la aplicación
                            echo "🔧 Verificando funcionalidad básica..."
                            if curl -s http://localhost:${DVWA_PORT}/setup.php | grep -q "setup"; then
                                echo "✅ Página de setup accesible"
                            else
                                echo "⚠️ Página de setup no accesible (puede ser normal si ya está configurada)"
                            fi
                            
                            # Verificar logs del contenedor
                            echo "📋 Últimos logs del contenedor DVWA:"
                            docker logs --tail 15 ${DVWA_CONTAINER}
                            
                            echo "📋 Últimos logs del contenedor MySQL:"
                            docker logs --tail 10 ${MYSQL_CONTAINER}
                            
                            # Verificar estado final de los contenedores
                            echo "📊 Estado final de los contenedores:"
                            docker ps --filter "name=${DVWA_CONTAINER}" --filter "name=${MYSQL_CONTAINER}" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
                            
                            echo "✅ Todas las verificaciones de salud completadas exitosamente"
                        '''
                    } catch (Exception e) {
                        echo "❌ Error en verificaciones de salud: ${e.getMessage()}"
                        echo "🔍 Ejecutando diagnóstico de emergencia..."
                        sh '''
                            echo "=== DIAGNÓSTICO DE EMERGENCIA ==="
                            echo "Timestamp: $(date)"
                            echo ""
                            
                            # Verificar estado del sistema
                            echo "💻 Estado del sistema:"
                            echo "- Memoria disponible: $(free -h | grep Mem | awk '{print $7}')"
                            echo "- Espacio en disco: $(df -h / | tail -1 | awk '{print $4}')"
                            echo "- Procesos Docker: $(pgrep dockerd | wc -l)"
                            echo ""
                            
                            # Intentar diagnóstico con manejo de errores
                            echo "🔍 Intentando diagnóstico de contenedores..."
                            if ./diagnose-containers.sh 2>&1; then
                                echo "✅ Diagnóstico completado"
                            else
                                echo "⚠️ Diagnóstico completado con errores"
                                echo ""
                                echo "🔧 Verificaciones alternativas:"
                                
                                # Verificar puertos en uso
                                echo "📡 Puertos en uso:"
                                netstat -tlnp 2>/dev/null | grep ":80\\|:3306\\|:9000" || echo "No se encontraron puertos relevantes"
                                echo ""
                                
                                # Verificar procesos relacionados
                                echo "🔄 Procesos relacionados:"
                                ps aux | grep -E "(docker|mysql|apache|nginx)" | grep -v grep || echo "No se encontraron procesos relevantes"
                                echo ""
                                
                                # Verificar conectividad básica
                                echo "🌐 Verificaciones de conectividad:"
                                if curl -s --connect-timeout 5 http://localhost:80/ >/dev/null 2>&1; then
                                    echo "✅ Puerto 80 responde"
                                else
                                    echo "❌ Puerto 80 no responde"
                                fi
                                
                                if curl -s --connect-timeout 5 http://localhost:9000/ >/dev/null 2>&1; then
                                    echo "✅ Puerto 9000 (SonarQube) responde"
                                else
                                    echo "❌ Puerto 9000 no responde"
                                fi
                            fi
                            
                            echo ""
                            echo "=== FIN DIAGNÓSTICO DE EMERGENCIA ==="
                        '''
                        
                        // Marcar como unstable en lugar de fallar completamente
                        currentBuild.result = 'UNSTABLE'
                        echo "⚠️ Las verificaciones de salud fallaron, pero el pipeline continuará como UNSTABLE"
                        echo "📋 Revise los logs de diagnóstico de emergencia para más detalles"
                    }
                }
            }
        }
        
        stage('Análisis Dinámico con OWASP ZAP') {
            steps {
                echo '🕷️ Iniciando análisis dinámico con OWASP ZAP...'
                script {
                    try {
                        sh '''
                            # Ejecutar escaneo ZAP con configuración mejorada
                            docker run --rm \\
                                --network ${ZAP_NETWORK} \\
                                -v $(pwd):/zap/wrk/:rw \\
                                ghcr.io/zaproxy/zaproxy:stable \\
                                zap-full-scan.py \\
                                -t http://${DVWA_CONTAINER}:80 \\
                                -r /zap/wrk/zap-report-${BUILD_TIMESTAMP}.html \\
                                -x /zap/wrk/zap-report-${BUILD_TIMESTAMP}.xml \\
                                -J /zap/wrk/zap-report-${BUILD_TIMESTAMP}.json \\
                                -m 10 \\
                                -z "-config scanner.strength=MEDIUM"
                            
                            echo "✅ Análisis dinámico completado"
                        '''
                    } catch (Exception e) {
                        echo "⚠️ Error en análisis dinámico: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Generar Reportes') {
            steps {
                echo '📊 Generando y archivando reportes...'
                script {
                    try {
                        sh '''
                            # Crear directorio de reportes
                            mkdir -p reports
                            
                            # Mover reportes a directorio organizado
                            mv zap-report-${BUILD_TIMESTAMP}.* reports/ 2>/dev/null || echo "No se encontraron algunos reportes ZAP"
                            
                            # Mover reportes de análisis estático básico
                            if [ -d "static-analysis" ]; then
                                mv static-analysis/* reports/ 2>/dev/null || echo "No se encontraron reportes de análisis estático"
                            fi
                            
                            # Generar reporte de estado del despliegue
                            cat > reports/deployment-report-${BUILD_TIMESTAMP}.txt << EOF
=== REPORTE DE DESPLIEGUE DVWA ===
Timestamp: ${BUILD_TIMESTAMP}
Build Number: ${BUILD_NUMBER}
Estado: $(docker ps --filter "name=${DVWA_CONTAINER}" --format "table {{.Status}}" | tail -n +2)

=== INFORMACIÓN DE CONTENEDORES ===
$(docker ps --filter "name=${DVWA_CONTAINER}" --filter "name=${MYSQL_CONTAINER}" --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}")

=== VERIFICACIONES DE SALUD ===
HTTP Status: $(curl -s -o /dev/null -w "%{http_code}" http://localhost:${DVWA_PORT}/ || echo "Error")
Aplicación Accesible: $(curl -s http://localhost:${DVWA_PORT}/ | grep -q "DVWA" && echo "✅ Sí" || echo "❌ No")

=== LOGS RECIENTES ===
$(docker logs --tail 5 ${DVWA_CONTAINER} 2>/dev/null || echo "No se pudieron obtener logs")
EOF
                            
                            # Listar archivos generados
                            echo "📁 Archivos de reporte generados:"
                            ls -la reports/
                        '''
                        
                        // Archivar artefactos
                        archiveArtifacts artifacts: 'reports/**/*', fingerprint: true, allowEmptyArchive: true
                        
                    } catch (Exception e) {
                        echo "⚠️ Error al generar reportes: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo '🧹 Ejecutando limpieza post-build...'
            script {
                try {
                    sh '''
                        # Limpiar imágenes no utilizadas (mantener las actuales)
                        docker image prune -f --filter "until=24h" || true
                        
                        # Mostrar estado final
                        echo "📊 Estado final de contenedores:"
                        docker ps --filter "name=${DVWA_CONTAINER}" --filter "name=${MYSQL_CONTAINER}"
                        
                        echo "💾 Uso de espacio en disco:"
                        df -h | grep -E "(Filesystem|/dev/)"
                    '''
                } catch (Exception e) {
                    echo "⚠️ Error en limpieza: ${e.getMessage()}"
                }
            }
        }
        
        success {
            echo '🎉 ¡Pipeline completado exitosamente!'
            script {
                sh '''
                    echo "✅ DVWA desplegado correctamente en http://localhost:${DVWA_PORT}"
                    echo "📊 Reportes disponibles en los artefactos del build"
                    echo "🔍 Análisis de seguridad completados"
                '''
            }
        }
        
        failure {
            echo '❌ Pipeline falló. Ejecutando limpieza de emergencia...'
            script {
                try {
                    sh '''
                        # Limpieza de emergencia
                        docker stop ${DVWA_CONTAINER} ${MYSQL_CONTAINER} || true
                        docker rm ${DVWA_CONTAINER} ${MYSQL_CONTAINER} || true
                        
                        # Mostrar logs para debugging
                        echo "🔍 Logs de debugging:"
                        docker logs ${DVWA_CONTAINER} --tail 20 2>/dev/null || echo "No se pudieron obtener logs de DVWA"
                        docker logs ${MYSQL_CONTAINER} --tail 20 2>/dev/null || echo "No se pudieron obtener logs de MySQL"
                    '''
                } catch (Exception e) {
                    echo "⚠️ Error en limpieza de emergencia: ${e.getMessage()}"
                }
            }
        }
        
        unstable {
            echo '⚠️ Build marcado como inestable. Verificar reportes de análisis.'
        }
    }
}


