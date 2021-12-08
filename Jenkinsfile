@Library('slack') _
pipeline {
  agent any
  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "thepuransingh/numeric-app:${GIT_COMMIT}"
    applicationURL = "http://devsecopshackathon2021.eastus.cloudapp.azure.com"
    applicationURI = "/increment/99"
  }
  
  stages {
      stage('Build Artifact - Maven') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' 
            }
        }  
        
  stage('Unit Tests - JUnit and Jacoco') {
      steps {
        sh "mvn test"
      }
    }
     
  stage('Mutation Tests - PIT') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }
   }
    
    stage('SonarQube - SAST') {
      steps {
        withSonarQubeEnv('SonarQube') {
        sh "mvn sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.host.url=http://devsecopshackathon2021.eastus.cloudapp.azure.com:9000"
      }
    	timeout(time: 2, unit: 'MINUTES') {
          script {
            waitForQualityGate abortPipeline: true
          }
        }
      }
    }
    
    
    //    stage('Vulnerability Scan - Docker ') {
    //      steps {
    //         sh "mvn dependency-check:check"   
    //        }
    // }
    
    stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
          "Dependency Scan": {
            sh "mvn dependency-check:check"
          },
          "Trivy Scan": {
            sh "bash trivy-docker-image-scan.sh"
          },
          "OPA Conftest": {
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
            }
        )
      }
    }
    
      stage('Docker Build and Push') {
      	steps {
	        withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
	          sh 'printenv'
	          sh 'sudo docker build -t thepuransingh/numeric-app:""$GIT_COMMIT"" .'
	          sh 'docker push thepuransingh/numeric-app:""$GIT_COMMIT""'
	        }
      	}
      }
      
    
    
   /* stage('Vulnerability Scan - Kubernetes') {
      steps {
        sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
      }
    } */
    
    stage('Vulnerability Scan - Kubernetes') {
      steps {
        parallel(
          "OPA Scan": {
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          },
          "Kubesec Scan": {
            sh "bash kubesec-scan.sh"
          },
          "Trivy Scan": {
            sh "bash trivy-k8s-scan.sh"
          }
        )
      }
    }
    
    /* if Rollout code not working then uncomment the below code */
    
    stage('Kubernetes Deployment - DEV') {
     steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh "sed -i 's#replace#thepuransingh/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
          sh "kubectl apply -f k8s_deployment_service.yaml"
        }
      }
    }
    
  
  /* This is rollout code, comment this if not working and uncomment the above code */ 
   
  /* stage('K8S Deployment - DEV') {
      steps {
        parallel(
          "Deployment": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment.sh"
            }
          },
          "Rollout Status": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment-rollout-status.sh"
            }
          }
        )
      }
    }
  } */
  
  
  
  
  
  
  /*---------- Promote to PROD? --------------- */
	  stage('Promote to PROD?') {
	  steps {
	    timeout(time: 2, unit: 'DAYS') {
	      input 'Do you want to Approve the Deployment to Production Environment/Namespace?'
	    }
	  }
	}
  
  
  stage('K8S Deployment - PROD') {
      steps {
        parallel(
          "Deployment": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "sed -i 's#replace#${imageName}#g' k8s_PROD-deployment_service.yaml"
              sh "kubectl -n prod apply -f k8s_PROD-deployment_service.yaml"
            }
          }		
          /*	,
          "Rollout Status": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-PROD-deployment-rollout-status.sh"
            }
          }
          */
          
        )
      }
    }
    
  /*---------- SLACK Notifications--------------- */
  
	  stage('Slack') {
	      steps {
	        sh 'exit 0'
	      }
	    }
	
 
  } 
  
    post {
    always {
      junit 'target/surefire-reports/*.xml'
      jacoco execPattern: 'target/jacoco.exec'
      pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
      // Use sendNotifications.groovy from shared library and provide current build result as parameter    
      sendNotification currentBuild.result
    }

    success {
      script {
        /* Use slackNotifier.groovy from shared library and provide current build result as parameter */
        env.failedStage = "none"
        env.emoji = ":white_check_mark: :tada: :thumbsup_all:"
        sendNotification currentBuild.result
      }
    }
    unstable {
      script {
        /* Use slackNotifier.groovy from shared library and provide current build result as parameter */
        env.failedStage = "none"
        env.emoji = ":uside_down_face: :thinking_face: :writing_hand:"
        sendNotification currentBuild.result
      }
    }
    

    // failure {

    // }
  }

}