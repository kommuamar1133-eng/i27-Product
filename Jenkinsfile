// This Jenkins file is for product Deployment
pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice(name: 'buildOnly',
               choices: 'no\nyes',           /// \n for next line
               description: 'This will only build the application'
        )
        choice(name: 'scanOnly',
               choices: 'no\nyes',           /// \n for next line
               description: 'This will only scan the application'
        )
        choice(name: 'dockerPush',
               choices: 'no\nyes',           /// \n for next line
               description: 'This will build the app, docker push' 
        )
        choice(name: 'deploytoDev',
               choices: 'no\nyes',           /// \n for next line
               description: 'This will deploy the app to Dev env' 
        )
        choice(name: 'deploytoTest',
               choices: 'no\nyes',           /// \n for next line
               description: 'This will deploy the app to Test env' 
        )
        choice(name: 'deploytoStage',
               choices: 'no\nyes',           /// \n for next line
               description: 'This will deploy the app to Stage env' 
        )
        choice(name: 'deploytoProd',
               choices: 'no\nyes',           /// \n for next line
               description: 'This will deploy the app to Prod env' 
        )
    }
    environment {
        APPLICATION_NAME = "product"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        // versioning + packaging
        DOCKER_HUB = "docker.io/amkommuaws123"
        DOCKERHUB_CREDS = credentials("DockerHub_Creds_amkommuaws123")
        SONAR_URL = "http://34.134.238.254:9000"
        SONAR_TOKEN = credentials("Sonar_Creds")
    }
    tools {
        maven "Maven-3.8.8"
        jdk "JDK-17"
    }
    stages {
        stage ('Build') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                        //params.dockerPush == 'yes'
                    }
                }
            }
            // Application Build happens here
            steps { //For jenkins env variables no need to write env.
                script {                    
                    buildApp().call()
                }
            }
        }
        stage ('Unit Tests') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                echo "Performing Unit tests for ${env.APPLICATION_NAME} application"
                sh "mvn test"
            }
            post {
                always {
                    junit "target/surefire-reports/*.xml"
                }
            }
        }
        stage ('Sonar') {
            when {
                expression {
                    params.scanOnly == 'yes'       
                }
            }    
            steps {
                echo "************ Starting SonarQube with Quality Gates **************"
                withSonarQubeEnv('SonarQube') { //manage jenkins > System > SonarQube Server
                    sh """
                    mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=i27-user \
                            -Dsonar.host.url=${env.SONAR_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                    """                
                }
                timeout (time: 2, unit: 'MINUTES') { //NANOSECONDS, SECONDS, MINUTES, HOURS, DAYS
                   script {
                       waitForQualityGate abortPipeline: true
                   }
                }   
            }
        }
        /*
        stage ('Docker Format') {
            steps {
                //Tell me how can i read pom.xml from jenkinsfile
                echo "Actual Format: ${env.APPLICATION_NAME}-${env.POM_VERSION}-${env.POM_PACKAGING}"
                // i Need to have below formating
                // user-buildNumber-branchName.packaging
                echo "Custom Format: ${env.APPLICATION_NAME}-${currentBuild.number}-${BRANCH_NAME}.${env.POM_PACKAGING}"
            }
        }*/
        stage ('Docker Build and Push') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                script {
                dockerBuildandPush().call()
                // docker build -t registryname:tag ./cicd
                // cp /home/amruthnation27/jenkins/workspace/i27-User_master/target/i27-user-0.0.1-SNAPSHOT.jar ./.cicd
                // workspace means before the target fldr : see below
                // workspace/target/i27-user-0.0.1-snapshot-jar
                // docker login docker.io -u username -p password
                // my docker should be like this: amkommuaws/user:tag
                }
            }
        }
        stage ('Deploy to Dev') {
            when {
                expression {
                    params.deploytoDev == 'yes'
                }
            }
            steps {
               script {
                   imageValidation().call()
                   dockerDeploy('dev', '5132', '8132').call()
                   echo "**********Deployed to Dev Succesfully*************"
               }
            }
        }
        stage ('Deploy to Test') {
            when {
                expression {
                    params.deploytoTest == 'yes'
                }
            }
            steps {
               script {
                  imageValidation().call()
                  echo "************* Entering Test Env *****************"
                  dockerDeploy('test', '6132', '8132').call()
               }
            }
        }
        stage ('Deploy to Stage') {
            when {
                expression {
                    params.deploytoStage == 'yes'
                }
            }
            steps {
                script {
                  imageValidation().call()
                  dockerDeploy('stage', '7132', '8132').call()
               }
            }
        }
        stage ('Deploy to Prod') {
            when {
               // deploytoProd ===> yes and branch should be "release/***"
               allOf {
                  anyOf {
                    expression {
                        params.deploytoProd == 'yes'
                    }
                  }
                  anyOf {
                    branch 'release/*'
                    // only tags with v.*.*.* should deploy to prod
                  }
               }
            }
            steps {
               timeout(time: 300, unit: 'SECONDS') { //5mins
                    input message: "Deploying to ${env.APPLICATION_NAME}-prod ???", ok: 'yes', submitter: "krish"
               }
               script {
                  imageValidation().call()
                  dockerDeploy('prod', '8132', '8132').call()
               }
            }
        }
        stage ('CleanWorkSpace') {
            steps {
                cleanWs()
            }
        }
    }
}

// This method will build image and push to registry
def dockerBuildandPush() {
    return {
        echo "************************* Building DOcker Image*************************"
        sh "cp ${workspace}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd"
        sh "ls -la ./.cicd"
        sh "docker build --force-rm --no-cache --pull --rm=true --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd"
        echo "************************* Login to Docker Repo*************************"
        sh "docker login -u ${DOCKERHUB_CREDS_USR} -p ${DOCKERHUB_CREDS_PSW}"
        echo "************************* Pushing Docker Image*************************"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        echo "********************* Pushed image successfull ********************"
    }
}


// This method is developed for deploying our application in different environments
def dockerDeploy(envDeploy, hostPort, contPort) {
    return {
    echo "************** Devploying to $envDeploy Environtment ******************"
    withCredentials([usernamePassword(credentialsId: 'amar_docker_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
        // some block
        // With the help of this block, the slave will be connecting to docker-vm and execute the commands to containers
        //sshpass -pfoobar ssh -o StrictHostKeyChecking=no user@host command_to_run 
        //sshpass -p password -v username@ipaddress command  ip=dockervm ipaddress
        //sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} hostname -i"
        
    script {
        // Pull the image on the Docker server
        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"                   
        
        try {
            // Stopping the Container
            echo "************** Stoping the Container ******************"
            sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop ${env.APPLICATION_NAME}-$envDeploy"
            
            // Remove the Container
            echo "*************** Removing the Container *****************"
            sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm ${env.APPLICATION_NAME}-$envDeploy"
        } catch(err) {
            echo "Caught the Error: $err"
          }
        // Create a container                                                                                             HostP:ContP 
        echo "**************** Creating the Container ******************"                                                                                                                          
        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p $hostPort:$contPort  --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
    }
    }
    }
}

def imageValidation() {
    return {
        println ("Pulling the docker image")
    try {
    sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
    }
    catch (Exception e) {
        println("OOPS!, docker images with this tag is not available")
        buildApp().call()
        dockerBuildandPush().call()
    }
    }
}


def buildApp() {
    return {
        echo "Building the ${env.APPLICATION_NAME} application"
        sh 'mvn clean package -DskipTests=true'
    }

}

// http://34.173.216.112:8080/sonarqube-webhook/


// User container runs at 8132 port (for Product)
// I will configure env's in a way they will have different host ports
// dev env   => 5132 (Host Port)
// test env  => 6132 (Host Port)
// stage env => 7132 (Host Port)
// dev env   => 8132 (Host Port)