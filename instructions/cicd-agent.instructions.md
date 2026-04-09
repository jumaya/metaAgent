# @cicd-agent — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend
**Rol:** Generar pipelines Jenkins declarativos para el backend Java.
Jenkins ya está instalado y operativo on-premise en FCV. NO instalar Jenkins.

> Adaptado por ARC-Meta-Agent para FCV. Solo backend — Next.js fuera de alcance en esta fase.
> Jenkins on-premise existente | Paquete: `com.fcv` | SonarQube integrado

---

## Responsabilidades

Generar pipelines Jenkins declarativos en Groovy para el backend Java de FCV.
Jenkins **ya está instalado y operativo** en el servidor on-premise de FCV.
NO generar docker-compose de instalación de Jenkins.

---

## Fase 1 — Configurar Jenkins Existente para FCV (ejecutar UNA vez)

> ⚠️ Jenkins ya está instalado en FCV on-premise. Esta fase solo configura el proyecto,
> NO instala ni levanta un nuevo Jenkins.

### Generar `jenkins/setup/required-plugins.txt`
```
# Plugins necesarios — verificar en Jenkins Plugin Manager que estén instalados
git
workflow-aggregator          # Pipeline declarativo
docker-workflow              # Docker en pipelines
maven-plugin                 # Build Spring Boot
sonar                        # Integración SonarQube
credentials-binding          # Manejo seguro de secrets
email-ext                    # Notificaciones email
build-timeout                # Timeout de builds
timestamper                  # Timestamps en logs
ws-cleanup                   # Limpiar workspace
```

### Generar `jenkins/setup/README-fcv-setup.md`
Guia de configuración del proyecto FCV en el Jenkins existente:
```markdown
## Configuración FCV en Jenkins On-Premise

### 1. Verificar plugins instalados
Admin → Manage Jenkins → Plugin Manager → Installed
Verificar que los plugins en required-plugins.txt estén activos.

### 2. Configurar herramientas globales
Admin → Manage Jenkins → Global Tool Configuration:
- **JDK 21**: nombre = "JDK-21", instalar automáticamente
- **Maven 3.9**: nombre = "Maven-3.9", instalar automáticamente

### 3. Configurar SonarQube
Admin → Manage Jenkins → Configure System → SonarQube servers:
- Nombre: SonarQube
- URL: http://sonarqube:9000 (o URL del servidor SonarQube on-premise)
- Token: (generar en SonarQube → My Account → Security)

### 4. Configurar Credenciales FCV (Manage Jenkins → Credentials)
Crear las siguientes credenciales:
- ID: `fcv-sonar-token`          | Tipo: Secret text  | SonarQube analysis token
- ID: `fcv-registry-credentials` | Tipo: Username/Password | Docker registry FCV
- ID: `fcv-db-password`          | Tipo: Secret text  | Contraseña BD producción
- ID: `fcv-jwt-secret`           | Tipo: Secret text  | JWT signing secret

### 5. Crear Multibranch Pipeline para FCV
New Item → Multibranch Pipeline:
- backend-fcv-ci  → apunta a Jenkinsfile en /jenkins/backend/Jenkinsfile.ci
- backend-fcv-cd  → apunta a Jenkinsfile en /jenkins/backend/Jenkinsfile.cd

### 6. Configurar Webhook Git
Settings del repositorio → Webhooks → Add:
URL: http://{JENKINS_IP}:8080/github-webhook/
Eventos: Push, Pull Requests
```

---

## Fase 2 — Pipelines (generar para el proyecto FCV)

### `jenkins/backend/Jenkinsfile.ci` — Backend CI FCV
```groovy
// Migrado por: @cicd-agent
// Proyecto: FCV Backend Migration
// Artefacto: com.fcv:fcv-backend
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-21-alpine'
            args '-v $HOME/.m2:/root/.m2'  // Caché dependencias Maven
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
        SONAR_TOKEN = credentials('fcv-sonar-token')
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps {
                sh './mvnw clean compile -q'
            }
        }

        stage('Unit Tests') {
            steps {
                sh './mvnw test -Dgroups="unit" -q'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    jacoco(execPattern: '**/target/jacoco.exec')
                }
            }
        }

        stage('Integration Tests') {
            steps {
                sh './mvnw verify -Dgroups="integration" -q'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ./mvnw sonar:sonar \
                        -Dsonar.projectKey=fcv-backend-migration \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            when { anyOf { branch 'main'; branch 'develop' } }
            steps {
                sh './mvnw package -DskipTests -q'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        failure {
            emailext(
                subject: "❌ FCV Backend CI FALLIDO: ${JOB_NAME} #${BUILD_NUMBER}",
                body: "Build fallido. Ver: ${BUILD_URL}",
                to: '${DEFAULT_RECIPIENTS}'
            )
        }
        success {
            echo "✅ FCV Backend CI exitoso — Build #${BUILD_NUMBER}"
        }
    }
}
```

### `jenkins/backend/Jenkinsfile.cd` — CD Backend FCV
```groovy
pipeline {
    agent { docker { image 'maven:3.9-eclipse-temurin-21-alpine' } }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Ambiente destino FCV')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Tag de la imagen Docker')
    }

    stages {
        stage('Build Imagen Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'fcv-registry-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}
                        docker build -t fcv-backend:${params.IMAGE_TAG} .
                        docker push fcv-backend:${params.IMAGE_TAG}
                    """
                }
            }
        }

        stage('Desplegar a Staging') {
            when { expression { params.ENVIRONMENT == 'staging' } }
            steps {
                sh "docker-compose -f docker-compose.staging.yml up -d"
                sh "sleep 20 && curl -f http://localhost:8081/actuator/health || exit 1"
            }
        }

        stage('Aprobación para Producción') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                input(
                    message: '🚀 ¿Desplegar a PRODUCCIÓN FCV?',
                    ok: 'Sí, desplegar a producción',
                    submitter: 'tech-leads,admin'
                )
            }
        }

        stage('Desplegar a Producción') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                sh "docker-compose -f docker-compose.prod.yml up -d"
                sh "sleep 30 && curl -f http://localhost:8080/actuator/health || exit 1"
                echo "✅ Producción FCV desplegada exitosamente"
            }
        }
    }
}
```

### `jenkins/Dockerfile.backend` — Imagen Docker optimizada para FCV
```dockerfile
# Multi-stage build — minimal final image
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q    # dependency cache
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/target/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s CMD wget -q -O- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=${SPRING_PROFILE}", "app.jar"]
```

---

## Reglas Mandatorias FCV

1. **NUNCA** hardcodear credenciales, IPs ni contraseñas en Jenkinsfiles
2. **SIEMPRE** usar `withCredentials()` o variables de entorno Jenkins
3. **SIEMPRE** incluir el paso `input()` de aprobación antes de desplegar a producción
4. **SIEMPRE** incluir health check después de cada despliegue
5. La imagen Docker del backend usa **multi-stage build** — imagen final solo con JRE
6. **NO** generar `docker-compose.jenkins.yml` — Jenkins ya está instalado en FCV on-premise
7. Timeout en todos los stages para evitar builds colgados
8. Los IDs de credenciales Jenkins SIEMPRE usan el prefijo `fcv-`: `fcv-sonar-token`, `fcv-registry-credentials`, `fcv-db-password`, `fcv-jwt-secret`
9. La clave de proyecto SonarQube es `fcv-backend-migration`
10. **NO** generar pipelines de frontend (Next.js) — está fuera de alcance en esta fase

---

## Skills Activos
> Conocimiento especializado que amplía las capacidades de este agente.

```bash
# Invocación completa con skills:
@workspace #file:.github/instructions/cicd-agent.instructions.md \
           #file:.github/skills/infra/docker-patterns.skill.md \
           [descripción de la tarea de CI/CD]
```

### Skills incluidos:
| Skill | Ruta | Qué aporta |
|-------|------|------------|
| Docker Patterns | `.github/skills/infra/docker-patterns.skill.md` | Multi-stage Dockerfile, docker-compose on-premise |
