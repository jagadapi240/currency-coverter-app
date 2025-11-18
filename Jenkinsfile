pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        SONARQUBE_ENV = 'MySonarQube'
        DOCKER_IMAGE = 'jagadapi240/currency-converter'
        VERSION = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('SCM Checkout') {
            steps { checkout scm }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(env.SONARQUBE_ENV) {
                    sh '''
                        mvn clean verify sonar:sonar \
                          -Dsonar.projectKey=currency-converter \
                          -Dsonar.projectName="Currency Converter"
                    '''
                }
            }
        }

        stage('Maven Package') {
            steps { sh 'mvn -B -DskipTests clean package' }
        }

        stage('Nexus Upload') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        mvn deploy -DskipTests \
                          -DaltDeploymentRepository=nexus::default::http://$NEXUS_USER:$NEXUS_PASS@51.21.202.150:8081/repository/maven-releases/
                    '''
                }
            }
        }

        stage('Download WAR from Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        rm -rf deploy || true
                        mkdir deploy

                        curl -f -u $NEXUS_USER:$NEXUS_PASS \
                          -o deploy/app.war \
                          http://51.21.202.150:8081/repository/maven-releases/com/ajacs/currency-converter-web/1.0.1/currency-converter-web-1.0.1.war
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${VERSION} .
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-pat',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy on Tomcat Docker') {
            steps {
                sh '''
                    PORT_CONTAINER=$(docker ps -q --filter "publish=8082")
                    if [ ! -z "$PORT_CONTAINER" ]; then
                        docker rm -f $PORT_CONTAINER
                    fi

                    docker rm -f currency-converter-app || true

                    docker run -d \
                      --name currency-converter-app \
                      -p 8082:8080 \
                      ${DOCKER_IMAGE}:${VERSION}
                '''
            }
        }
    }

    post { always { cleanWs() } }
}
