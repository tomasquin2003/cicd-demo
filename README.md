
# CICD-DEMO

This project aims to be the basic skeleton to apply continuous integration and continuous delivery.

## Topology

CICD Demo uses some kubernetes primitives to deploy:

* Deployment
* Services
* Ingress ( with TLS )

```bash
     internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
   --|-----|--
   [   Pods   ]

```

This project includes:

* Spring Boot java app
* Jenkinsfile integration to run pipelines
* Dockerfile containing the base image to run java apps
* Makefile and docker-compose to make the pipeline steps much simpler
* Kubernetes deployment file demonstrating how to deploy this app in a simple Kubernetes cluster

## Pipeline Setup (Taller Local CI/CD)

Este proyecto se ha configurado para correr en un entorno local controlado utilizando Docker. Sigue estos pasos detalladamente para completar el taller.

### 1. Preparar la Red y los Servicios

Para que Jenkins y SonarQube se comuniquen correctamente por nombre de host, deben estar en la misma red de Docker.

```bash
# Crear red de Docker
docker network create cicd-network

# Levantar SonarQube
docker run -d --name sonarqube-local --network cicd-network -p 9000:9000 sonarqube:lts-community

# Levantar Jenkins (mapeando el Docker socket del host)
docker run -d --name jenkins-local --network cicd-network -p 8080:8080 -p 50000:50000 -v /var/run/docker.sock:/var/run/docker.sock -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

### 2. Configurar Jenkins

Accede a `http://localhost:8080`. Se requieren los siguientes plugins instalados:
- **Pipeline**
- **Git**
- **Maven Integration**
- **SonarQube Scanner for Jenkins**
- **Docker Pipeline**
- **Workspace Cleanup**

#### Configuración de Herramientas:
1. **Maven**: Ve a **Manage Jenkins** > **Tools**. En la sección **Maven installations**, añade una versión (ej. 3.9.x) con el nombre exacto **`maven-3`**.
2. **SonarQube**: 
   - Ve a **Manage Jenkins** > **System**. En **SonarQube servers**, añade uno con:
     - Name: **`SonarQube`**
     - Server URL: **`http://sonarqube-local:9000`**
   - Obtén un Token en SonarQube (`My Account` > `Security` > `Tokens`) y regístralo en Jenkins como credencial de tipo "Secret text".

#### Configuración de Webhook en SonarQube:
Es vital para que la etapa de `Quality Gate` no se quede esperando infinitamente:
1. En Sonarqube, ve a **Administration** > **Configuration** > **Webhooks**.
2. Crea un Webhook:
   - Name: `Jenkins-Webhook`
   - URL: **`http://jenkins-local:8080/sonarqube-webhook/`**

### 3. Crear el Pipeline

1. Crea un nuevo item de tipo **Pipeline**.
2. En la sección **Pipeline**, selecciona **Pipeline script from SCM**.
3. Selecciona **Git** e ingresa la URL de este repositorio.
4. Asegúrate que **Script Path** sea **`Jenkinsfile`**.
5. Ejecuta con **Build Now**.

### 4. Validación
Tras un despliegue exitoso, la app estará en `http://localhost:80`. El escaneo de seguridad (Trivy) se ejecuta vía Docker, por lo que no necesitas instalarlo localmente.

---

## Testing

Unit tests and integrations tests are separated using [JUnit Categories][].

[JUnit Categories]: https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit.html

### Unit Tests

```java
mvn test -Dgroups=UnitTest
```

Or using Docker:

```bash
make build
```

### Integration Tests

```java
mvn integration-test -Dgroups=IntegrationTests
```

Or using Docker:

```bash
make integrationTest
```

### System Tests

System tests run with Selenium using docker-compose to run a [Selenium standalone container][] with Chrome.

[Selenium standalone container]: https://github.com/SeleniumHQ/docker-selenium

Using Docker:

* If you are running locally, make sure the `$APP_URL` is populated and points to a valid instance of your application. This variable is populated automatically in Jenkins.

```bash
APP_URL=http://dev-cicd-demo-master.anzcd.internal/ make systemTest
```