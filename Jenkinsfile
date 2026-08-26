pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'maven-devsecops'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/abhradippaul/Boardgame.git'
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
            steps {
                sh "docker run --rm -v /data/trivy/fs:/data -v /var/run/docker.sock:/var/run/docker.sock -v ~/.cache:/root/.cache --name trivy aquasec/trivy fs --format table -o /data/BoardGame.html ."
                echo 'File System Scan Completed'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("sonar-scanner") {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectKey=BoardGame \
                    -Dsonar.projectName=BoardGame \
                    -Dsonar.java.binaries=. 
                '''
                }
                echo 'SonarQube Analysis Completed'
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                    echo "Quality gate passed successfully"
                }
            }
        }

        stage("Build") {
            steps {
                sh "mvn package"
                echo "Maven build successfully"
            }
        }
        
        stage('Publish to Nexus') {
            steps {
              withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
                echo "Publish to nexus is completed."
            }
        }
        
        stage('Login to DockerHub and Build Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'jenkins-docker', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "docker build -t ${username}/${DOCKER_IMAGE}:${params.BUILD_NUMBER} ."
                    sh "docker login -u ${username} -p ${password}"
                }
                echo "Docker Image build and Docker Hub login completed."
            }
        }
        
        stage('Scan Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'jenkins-docker', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "docker run --rm -v /data/trivy/image:/data -v /var/run/docker.sock:/var/run/docker.sock -v ~/.cache:/root/.cache --name trivy aquasec/trivy image --format json --output /data/BoardGame.json ${username}/${DOCKER_IMAGE}:${params.BUILD_NUMBER}"
                }
                echo "Scan Docker Image completed."
            }
        }
        
        stage('Publish to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'jenkins-docker', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "docker push ${username}/${DOCKER_IMAGE}:${params.BUILD_NUMBER}"
                }
                echo "Publish to Docker Hub is completed."
            }
        }
}
