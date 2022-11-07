pipeline{
    agent any 
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
            
            steps{
                script{
                    withSonarQubeEnv('sonarqube') {
                            sh 'chmod +x gradlew'
                            sh './gradlew sonarqube'
                    }

                    timeout(time: 1, unit: 'HOURS') {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }

                }  
            }
        }
        
        stage("docker build & docker push"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker-pass', variable: 'docker_password')]) {
                             sh '''
                                docker build -t 10.162.0.6:8083/springapp:${VERSION} .
                                docker login -u admin -p $docker_password 10.162.0.6:8083 
                                docker push  10.162.0.6:8083/springapp:${VERSION}
                                docker rmi 10.162.0.6:8083/springapp:${VERSION}
                            '''
                    }
                }
            }
        }
        
        stage('indentifying misconfigs using datree in helm charts'){
            steps{
                script{

                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=0e913831-aa79-453a-82a7-e59c6512620a']) {
                            sh 'helm datree test myapp/'
                        }
                    }
                }
            }
        }
        
        stage("pushing the helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker-pass', variable: 'docker_password')]) {
                          dir('kubernetes/') {
                             sh '''
                                 helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                 tar -czvf  myapp-${helmversion}.tgz myapp/
                                 curl -u admin:$docker_password http://10.162.0.6:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                          }
                    }
                }
            }
        }
        
	stage('approval') {
		steps {
			script {
				timeout(time: 1, unit: 'DAYS') {
   					 input message: 'do u wan to deploy of kube cluster?', ok: 'yes go deploy', submitter: 'admin'
				}
			}
		}	
	}
        stage('deploy to k') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubeconfig_file', namespace: '', serverUrl: '') {
                        dir('kubernetes/') {
			                sh 'helm upgrade --install --set image.repository="10.162.0.6:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ '
			}
		    }			 
                }
            }
        }
	    
	stage('email') {
            steps {
                script {
            	     mail bcc: '', body: '<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL to build: ${env.BUILD_URL}', cc: '', from: '', replyTo: '', subject: '${BUILD_ID}', to: 'mm.moghal007@gmail.com'  
                }
    	    }
	}
    }
}
