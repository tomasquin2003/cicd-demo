#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        maven 'maven-3'
    }

    environment {
        APP_NAME = "cicd-demo-app"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test -DexcludedGroups=au.com.equifax.cicddemo.domain.SystemTest'
            }
        }

        stage('Static Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "mvn sonar:sonar -Dsonar.projectKey=${APP_NAME}"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${APP_NAME}:latest ."
            }
        }

        stage('Container Security Scan') {
            steps {
                sh """
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy image \
                  --timeout 20m \
                  --scanners vuln \
                  --exit-code 1 \
                  --severity CRITICAL \
                  ${APP_NAME}:latest
                """
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                docker rm -f cicd-app || true
                docker run -d --name cicd-app -p 80:8080 ${APP_NAME}:latest
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline ejecutado exitosamente.'
        }
        failure {
            echo 'El Pipeline falló. Revisa los logs para más detalles.'
        }
    }
}
