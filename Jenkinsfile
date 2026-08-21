pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        DOCKER_IMAGE = 'boardgame'
        IMAGE_TAG = "${BUILD_NUMBER}"
        NEXUS_REGISTRY = '192.168.176.128:8082'
    }

    stages {

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar \
                      -Dsonar.projectKey=Boardgame \
                      -Dsonar.projectName=Boardgame
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Scan') {
            environment {
                TMPDIR = "${WORKSPACE}/trivy-tmp"
            }
            steps {
                sh 'mkdir -p $TMPDIR'
                sh """
                trivy image --severity HIGH,CRITICAL \
                --cache-dir ${WORKSPACE}/trivy-cache \
                --timeout 15m \
                --format table \
                -o trivy-report.txt \
                ${DOCKER_IMAGE}:${IMAGE_TAG}
                """
            }
        }

        stage('Push to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                    docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${NEXUS_REGISTRY}/${DOCKER_IMAGE}:${IMAGE_TAG}
                    echo \$NEXUS_PASS | docker login ${NEXUS_REGISTRY} -u \$NEXUS_USER --password-stdin
                    docker push ${NEXUS_REGISTRY}/${DOCKER_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
        }
    }
}
