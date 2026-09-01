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

        stage('Git Checkout') {
            steps {
                checkout scm
                echo "Git Checkout completed successfully."
            }
        }

        stage('Compilation Code') {
            steps {
                sh "mvn compile"
                echo "Compilation is done"
            }
        }

        stage('Test Compilation') {
            steps {
                sh "mvn test"
                echo "Test Compilation is done"
            }
        }

        stage('File System Scan') {
            environment {
                TMPDIR = "${WORKSPACE}/trivy-tmp"
            }
            steps {
                sh 'mkdir -p $TMPDIR'
                sh """
                trivy fs --severity HIGH,CRITICAL \
                --cache-dir /var/lib/jenkins/trivy-cache \
                --timeout 30m \
                --format table \
                -o trivy-fs-report.txt \
                .
                """
                echo 'File System Scan Completed'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("SonarQube") {
                    sh '''
                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar \
                      -Dsonar.projectKey=Boardgame \
                      -Dsonar.projectName=Boardgame
                    '''
                }
                echo 'SonarQube Analysis Completed'
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
                echo "Quality gate passed successfully"
            }
        }

        stage("Maven Package") {
            steps {
                sh "mvn package"
                echo "Maven build successfully"
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
                echo "Docker Image build completed."
            }
        }

        stage('Publish to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                    docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${NEXUS_REGISTRY}/${DOCKER_IMAGE}:${IMAGE_TAG}
                    docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${NEXUS_REGISTRY}/${DOCKER_IMAGE}:latest
                    echo \$NEXUS_PASS | docker login ${NEXUS_REGISTRY} -u \$NEXUS_USER --password-stdin
                    docker push ${NEXUS_REGISTRY}/${DOCKER_IMAGE}:${IMAGE_TAG}
                    docker push ${NEXUS_REGISTRY}/${DOCKER_IMAGE}:latest
                    """
                }
                echo "Publish to Nexus is completed."
            }
        }

        stage('Scan Docker Image') {
            environment {
                TMPDIR = "${WORKSPACE}/trivy-tmp"
            }
            steps {
                sh 'mkdir -p $TMPDIR'
                sh """
                trivy image --severity HIGH,CRITICAL \
                --cache-dir /var/lib/jenkins/trivy-cache \
                --timeout 30m \
                --format table \
                -o trivy-image-report.txt \
                ${DOCKER_IMAGE}:${IMAGE_TAG}
                """
                echo "Scan Docker Image completed."
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f deployment.yaml"
                sh "kubectl apply -f service.yaml"
                sh "kubectl rollout status deployment/boardgame-deployment"
                echo "Deployment to Kubernetes completed."
            }
        }
    }

    post {
    	always {
        	archiveArtifacts artifacts: 'trivy-*-report.txt', allowEmptyArchive: true
    	}
    	failure {
        	mail to: 'TON_EMAIL@gmail.com',
             	subject: "ÉCHEC Pipeline Jenkins : ${JOB_NAME} - Build #${BUILD_NUMBER}",
             	body: """
             	Le pipeline a échoué.
             
             	Job: ${JOB_NAME}
             	Build: #${BUILD_NUMBER}
             	Voir les détails: ${BUILD_URL}console
             	"""
    	}
    	success {
        	mail to: 'ranimgouissem657@gmail.com',
             	subject: "SUCCÈS Pipeline Jenkins : ${JOB_NAME} - Build #${BUILD_NUMBER}",
             	body: "Le pipeline s'est terminé avec succès ! Voir: ${BUILD_URL}"
    	}
    }
}

