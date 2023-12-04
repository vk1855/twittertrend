def registry = 'https://vk1855.jfrog.io'
def imageName = 'vk1855.jfrog.io/vk1855-docker-local/ttrend'
def version   = '2.1.2'

pipeline {
    agent {
        node {
            
            label 'maven'
        }
    }

environment {
    PATH = "/opt/apache-maven-3.9.6/bin:$PATH"
}

    stages{
       stage ("build"){
         steps {
            echo "-------------------Build Started-------------------------"
            sh 'mvn clean deploy -Dmaven.test.skip=true'
            echo "-------------------Build Completed-------------------------"
         }
       }

       stage("test"){
          steps{
             echo "-------------------Unit Test Started-------------------------"
             sh 'mvn surefire-report:report' 
             echo "-------------------Unit Test completed-------------------------"
          }
       }

       stage('SonarQube analysis') {
          environment {
            scannerHome = tool 'vk1855-sonar-scanner'
          }
         
         steps{
          withSonarQubeEnv('vk1855-sonarqube-server') { // If you have configured more than one global server connection, you can specify its name
          sh "${scannerHome}/bin/sonar-scanner"
         }        
    }
  }

       stage("Quality Gate"){
             steps{
                script{
                    timeout(time: 1, unit: 'HOURS') { // Just in case something goes wrong, pipeline will be killed after a timeout
                    def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                    if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
                }
             }
        }  


        stage("Jar Publish") {
        steps {
            script {
                    echo '<--------------- Jar Publish Started --------------->'
                     def server = Artifactory.newServer url:registry+"/artifactory" ,  credentialsId:"jfrog-cred"
                     def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                     def uploadSpec = """{
                          "files": [
                            {
                              "pattern": "jarstaging/(*)",
                              "target": "mvn-vk1855-libs-release-local/{1}",
                              "flat": "false",
                              "props" : "${properties}",
                              "exclusions": [ "*.sha1", "*.md5"]
                            }
                         ]
                     }"""
                     def buildInfo = server.upload(uploadSpec)
                     buildInfo.env.collect()
                     server.publishBuildInfo(buildInfo)
                     echo '<--------------- Jar Publish Ended --------------->'  
            
            }
        }   
    }  



         stage(" Docker Build ") {
          steps {
            script {
               echo '<--------------- Docker Build Started --------------->'
               app = docker.build(imageName+":"+version)
               echo '<--------------- Docker Build Ends --------------->'
           }
        }
      }

        stage (" Docker Publish "){
          steps {
             script {
                 echo '<--------------- Docker Publish Started --------------->'  
                    docker.withRegistry(registry, 'jfrog-cred'){
                    app.push()
                }    
                 echo '<--------------- Docker Publish Ended --------------->'  
            }
        }
    } 

}

}
