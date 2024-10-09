pipeline {

  agent any
  
  environment {
    my_registry = "ingcloudfr/vprofileapp-k8s"
    my_registry_credentials = "ingcloudfr-dockerhub"
    SONARSCANNER = 'sonarscanner4.7' // SonarQube scanner pour analyse de code
    SONARSERVER = 'my-sonar-server'  // Serveur SonarQube pour analyse
    DOCKER_IMAGE_TAG = "V${BUILD_NUMBER}"
    FULL_IMAGE_NAME = "${my_registry}:${DOCKER_IMAGE_TAG}"
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

    stage('Upload image on dockerhub') {
      steps {
        script {
          docker.withRegistry('', my_registry_credentials) {
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
      echo 'Cleaning up...'
      sh 'docker system prune -f'
    }

    success {
      echo "Pipeline completed successfully: ${env.BUILD_URL}"
    }

    failure {
      echo "Pipeline failed: ${env.BUILD_URL}"
      // Example of Slack notification in case of failure
      slackSend(channel: '#jenkins', color: 'danger', message: "Pipeline failed: ${env.BUILD_URL}")
    }
  }
}
