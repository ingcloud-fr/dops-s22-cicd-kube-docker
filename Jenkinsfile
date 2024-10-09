def COLOR_MAP = [
  'SUCCESS': 'good',  // Couleur verte pour succès dans la notification Slack
  'FAILURE': 'danger', // Couleur rouge pour échec dans la notification Slack
]

pipeline {

  agent any
  
  environment {
    MY_REGISTRY = "ingcloudfr/vprofileapp-k8s"                    // Nom du registre Docker
    MY_REGISTRY_CREDENTIALS = "ingcloudfr-dockerhub"              // Identifiant des credentials pour DockerHub
    SONARSCANNER = 'sonarscanner4.7'                              // SonarQube scanner pour analyse de code
    SONARSERVER = 'my-sonar-server'                               // Serveur SonarQube pour analyse
    DOCKER_IMAGE_TAG = "V${BUILD_NUMBER}"                         // Tag Docker basé sur le numéro de build
    FULL_IMAGE_NAME = "${MY_REGISTRY}:${DOCKER_IMAGE_TAG}"        // Nom complet de l'image Docker avec tag
    SLACK_CHANNEL = '#jenkins-cicd'                               // Canal Slack pour les notifications
  }

  stages {

    stage('Build with Maven') {
      steps {
        sh 'mvn clean install -DskipTests'
      }
      post {
        success {
          echo 'Now Archiving...'
          archiveArtifacts artifacts: '**/target/*.war'
        }
      }
    }

    stage('Unit test') {
      steps {
        sh 'mvn test'
      }
      post {
        unsuccessful {
          echo 'Unit tests failed'
          error 'Stopping pipeline due to failing tests'
        }
      }
    }

    stage('Integration tests') {
      steps {
        sh 'mvn verify -DskipUnitTests'
      }
      post {
        unsuccessful {
          echo 'Integration tests failed'
          error 'Stopping pipeline due to failing tests'
        }
      }
    }

    stage('Code analysis with mvn checkstyle') {
      steps {
        sh 'mvn checkstyle:checkstyle'
      }
      post {
        success {
          echo 'Generated Checkstyle Analysis Result'
        }
      }
    }

    stage('Code analysis with SonarQube') {

      environment {
        scannerHome = tool "${SONARSCANNER}"
      }

      steps {
        withSonarQubeEnv("${SONARSERVER}") {
          sh '''
            ${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
            -Dsonar.projectName=vprofile-repo \
            -Dsonar.projectVersion=1.0 \
            -Dsonar.sources=src/ \
            -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
            -Dsonar.junit.reportsPath=target/surefire-reports/ \
            -Dsonar.jacoco.reportsPath=target/jacoco.exec \
            -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
          '''
        }

        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Docker App image') {
      steps {
        script {
          dockerImage = docker.build(FULL_IMAGE_NAME)
        }
      }
    }

    stage('Upload image on DockerHub') {
      steps {
        script {
          docker.withRegistry('', MY_REGISTRY_CREDENTIALS) {
            dockerImage.push(DOCKER_IMAGE_TAG)
            dockerImage.push("latest")
          }
        }
      }
    }

    stage('Remove local unused docker images') {
      steps {
        sh 'docker image prune -f'
      }
    }

    stage('Kubernetes Deploy') {
      agent {
        label 'kops'
      }
      steps {
        sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${FULL_IMAGE_NAME} --namespace prod"
      }
    }
  }

  post {
    always {
      // Nettoyage Docker
      echo 'Cleaning up...'
      sh 'docker system prune -f'
      // Notifications Slack après l'exécution du pipeline (réussite ou échec)
      echo 'Slack Notifications'
      slackSend channel: "${SLACK_CHANNEL}", // Envoi d'une notification dans le canal Slack
      color: COLOR_MAP[currentBuild.currentResult], // Couleur basée sur le résultat du build
      message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}" // Message avec les détails du job
    }

    success {
      echo "Pipeline completed successfully: ${env.BUILD_URL}"
    }

    failure {
      echo "Pipeline failed: ${env.BUILD_URL}"
      // Exemple de notification Slack en cas d'échec
      slackSend(channel: '#jenkins', color: 'danger', message: "Pipeline failed: ${env.BUILD_URL}")
    }
  }
}
