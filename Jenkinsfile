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

        stage('1. GitHub Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Maven Compile & Test') {
            steps {
                sh 'mvn compile'
                sh 'mvn test'
            }
        }

        stage('3. Trivy FS Scan') {
            environment {
                TMPDIR = "${WORKSPACE}/trivy-tmp"
            }
            steps {
                sh 'mkdir -p $TMPDIR'
                sh """
                trivy fs --severity HIGH,CRITICAL \
                --cache-dir /var/lib/jenkins/trivy-cache \
                --timeout 15m \
                --format table \
                -o trivy-fs-report.txt \
                .
                """
            }
        }

        stage('4. SonarQube Analysis') {
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

        stage('5. Maven Package') {
            steps {
                sh 'mvn package'
            }
        }

        stage('6. Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
            }
        }

        stage('7. Push to Nexus') {
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

        stage('8. Trivy Image Scan') {
            environment {
                TMPDIR = "${WORKSPACE}/trivy-tmp"
            }
            steps {
                sh 'mkdir -p $TMPDIR'
                sh """
                trivy image --severity HIGH,CRITICAL \
                --cache-dir /var/lib/jenkins/trivy-cache \
                --timeout 15m \
                --format table \
                -o trivy-image-report.txt \
                ${NEXUS_REGISTRY}/${DOCKER_IMAGE}:${IMAGE_TAG}
                """
            }
        }

        stage('9. Docker Tag Latest') {
            steps {
                sh "docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest"
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-*-report.txt', allowEmptyArchive: true
        }
    }
}

