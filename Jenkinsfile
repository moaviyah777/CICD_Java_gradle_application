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
    }
}
