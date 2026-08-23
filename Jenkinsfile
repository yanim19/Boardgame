pipeline {
    agent any

    tools{
	maven 'maven3'
	jdk 'jdk17'
    }	    
    environment {
        // Outils & Serveurs
        SCANNER_HOME = tool 'SonarQubeScanner'
        NEXUS_URL = '192.168.1.50:8081' 
        DOCKER_REGISTRY = '192.168.1.50:8082' // Port du registre Docker Nexus
        CREDENTIALS_ID = 'nexus-credentials'
        
        // Application & Docker
        IMAGE_NAME = 'my-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        FULL_IMAGE_PATH = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        
        // Notifications
        NOTIF_EMAIL = 'devops-team@example.com'
        SLACK_CHANNEL = '#alerts-devsecops'
    }

    stages {
        stage('1. Code Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('2. Maven Compile') {
            steps {
                // Étape essentielle avant les scans : compilation du code
                sh 'mvn clean compile'
            }
        }

        stage('3. Trivy FS Scan (FAIL-FAST)') {
            steps {
                // Analyse ultra-rapide des dépendances (pom.xml) et des secrets.
                // Le pipeline s'arrête NET ici en cas de faille HIGH ou CRITICAL.
		timeout(time: 15, unit: 'MINUTE'){
                      sh "trivy fs --exit-code 1 --severity HIGH,CRITICAL ."
            	}
	    }	        
	 }

        stage('4. SonarQube Analysis') {
            steps {
                // Exécuté uniquement si Trivy a validé la sécurité du projet
                withSonarQubeEnv('SonarQube-Server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=my-devsecops-app \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=." 
                }
            }
        }

        stage('5. Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('6. Maven Package') {
            steps {
                // Génération du livrable final (JAR/WAR)
                sh 'mvn package -DskipTests'
            }
        }

        stage('7. Push Artifact to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUS_URL}",
                    groupId: 'com.example',
                    version: "${IMAGE_TAG}",
                    repository: 'maven-releases',
                    credentialsId: "${CREDENTIALS_ID}",
                    artifacts: [[artifactId: "${IMAGE_NAME}", file: 'target/my-app.war', type: 'war']]
                )
            }
        }

        stage('8. Build & Push Docker Image') {
            steps {
                script {
                    // Construction de l'image contenant le fichier .war
                    dockerImage = docker.build("${FULL_IMAGE_PATH}")
                    
                    // Envoi vers le registre privé Nexus
                    docker.withRegistry("http://${DOCKER_REGISTRY}", "${CREDENTIALS_ID}") {
                        dockerImage.push()
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs() // Nettoyage de l'espace de travail
        }
        success {
            // Notification en cas de succès général
            slackSend channel: "${SLACK_CHANNEL}", color: '#00FF00', message: "SUCCESS: Le pipeline ${BUILD_NUMBER} de ${JOB_NAME} s'est terminé avec succès !"
        }
        failure {
            // Notification immédiate par Email et Slack en cas d'échec (Trivy, Sonar, Nexus...)
            mail to: "${NOTIF_EMAIL}",
                 subject: "ÉCHEC Pipeline Jenkins : ${JOB_NAME} - Build #${BUILD_NUMBER}",
                 body: "Le pipeline a échoué. Veuillez vérifier les logs de la console Jenkins : ${BUILD_URL}"
                 
            slackSend channel: "${SLACK_CHANNEL}", color: '#FF0000', message: "FAILURE: Le pipeline ${BUILD_NUMBER} de ${JOB_NAME} a échoué. Vérifiez les outils de scan ou l'infrastructure : ${BUILD_URL}"
        }
    }
}

