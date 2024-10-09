pipeline {

	agent any
	/*
	tools {
	    maven "maven3"
	}
	*/
	environment {
    my_registry = "ingcloudfr/vprofileapp-k8s"
    my_registry_credentials = "ingcloudfr-dockerhub"
    SONARSCANNER = 'sonarscanner4.7' // SonarQube scanner pour analyse de code
    SONARSERVER = 'my-sonar-server' // Serveur SonarQube pour analyse
	  ARTVERSION = "${env.BUILD_ID}"
	}

	stages {

		stage('BUILD') {
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

		stage('UNIT TEST') {
			steps {
				sh 'mvn test'
			}
		}

		stage('INTEGRATION TEST') {
			steps {
				sh 'mvn verify -DskipUnitTests'
			}
		}

		stage('CODE ANALYSIS WITH CHECKSTYLE') {
			steps {
				sh 'mvn checkstyle:checkstyle'
			}
			post {
				success {
					echo 'Generated Analysis Result'
				}
			}
		}

		stage('CODE ANALYSIS with SONARQUBE') {

			environment {
				scannerHome = tool "${SONARSCANNER}"
			}

			steps {
				withSonarQubeEnv("${SONARSERVER}") {
					sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
				   -Dsonar.projectName=vprofile-repo \
				   -Dsonar.projectVersion=1.0 \
				   -Dsonar.sources=src/ \
				   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
				   -Dsonar.junit.reportsPath=target/surefire-reports/ \
				   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
				   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
				}

				timeout(time: 10, unit: 'MINUTES') {
					waitForQualityGate abortPipeline: true
				}
			}
		}

    stage('Build Docker App image') {
      steps {
        script {
          dockerImage = docker.build my_registry + ":V$BUILD_NUMBER" 
        }
      }
    }

    stage('Upload image on dockerhub'){
      steps {
        script {
          docker.withRegistry('', my_registry_credentials) {
            dockerImage.push("V$BUILD_NUMBER")
            dockerImage.push("latest")
          }
        }
      }
    }

    stage('Remove local unused docker images'){
      steps {
        sh 'docker rmi $my_registry:V$BUILD_NUMBER'
      }
    }

    stage('')

	}

}
