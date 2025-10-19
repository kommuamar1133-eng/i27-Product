// This Jenkinsfile is for product Deployment

pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice(name: 'scanOnly',
            choices: ['no', 'yes'],
            description: 'This will scan your application'
        )
        choice(name: 'buildOnly',
            choices: ['no', 'yes'],
            description: 'This will only build your application'
        )
        choice(name: 'dockerPush',
            choices: ['no', 'yes'],
            description: 'This will build dockerImage and push'
        )
        choice(name: 'deployToDev',
            choices: ['no', 'yes'],
            description: 'This will only Deploy the app to Dev env'
        )
        choice(name: 'deployToTest',
            choices: ['no', 'yes'],
            description: 'This will only Deploy the app to Test env'
        )
        choice(name: 'deployToStage',
            choices: ['no', 'yes'],
            description: 'This will only Deploy the app to stage env'
        )
        choice(name: 'deployToProd',
            choices: ['no', 'yes'],
            description: 'This will only Deploy the app to Prod env'
        )
    }
    tools {
        maven 'Maven-3.9.11'
        jdk 'JDK-17'
    }
    environment {
        APPLICATION_NAME = "product"
        SONAR_TOKEN = credentials('sonar_creds')
        SONAR_URL = "http://34.46.97.238:9000"
        // https://www.jenkins.io/doc/pipeline/steps/pipeline-utility-steps/#readmavenpom-read-a-maven-project-file
        // If any errors with readMavenPom, make sure pipeline-utility-steps plugin is installed in your jenkins, if not do install it
        // Script Approval issues : http://34.148.12.185:8080/scriptApproval/
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_HUB = "docker.io/kommuamar1133"
        DOCKER_CREDS = credentials('dockerhub_creds')
    }
    //Stages
    stages {
        stage ('Build') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                        params.buildOnly == 'yes'
                    }
                }
            }
            steps {
                script {
                    buildApp().call()
                }
            }
        }
        stage ('Sonar') {
            when {
                expression {
                    params.scanOnly == 'yes'
                    // params.buildOnly == 'yes'
                    // params.dockerPush == 'yes'
                }
            }
            steps {
                script {
                    sonar().call()
                }
            }
        }
        stage ('Docker Build & Push') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                script {
                    dockerBuildAndPush().call()
                }
            }

        }
        stage ('Deploy to Dev-Server') {
            when {
                anyOf {
                    expression {
                        params.deployToDev == 'yes'
                    }
                }
            }
            steps {
                script {
                    // envDeploy, hostPort, contPort
                    imageValidation().call()
                    dockerDeploy('dev', '5132', '8132').call()
                }
            }
        }
        stage ('Deploy to Test-Server') {
            when {
                anyOf {
                    expression {
                        params.deployToTest == 'yes'
                    }
                }
            }
            steps {
                script {
                    // envDeploy, hostPort, contPort
                    imageValidation().call()
                    dockerDeploy('tst', '6132', '8132').call()
                }       
            }
        }
        stage ('Deploy to Stage-Server') {
            when {
                allOf {
                    anyOf {
                        expression {
                            params.deployToStage == 'yes'
                            //other condition
                        }
                    }
                    anyOf {
                        branch 'release/*'
                    }
                }
            }
            steps {
                script {
                    // envDeploy, hostPort, contPort
                    imageValidation().call()
                    dockerDeploy('stg', '7132', '8132').call()
                }         
            }
        }
        stage ('Deploy to Prod-Server') {
            // Make sure only tags are deployed?
            when {
                allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                        }
                    }
                    anyOf {
                        tag pattern: "v\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}", comparator: "REGEXP"  //v1.2.3
                    }
                }
            }
            steps { 
                timeout(time: 300 , unit: 'SECONDS' ) {  //SECONDS, MINUTES, HOURS
                    input message: "Deploying to ${APPLICATION_NAME} to production ??", ok: 'yes', submitter: 'nanisre'
                }  
                script {
                    // envDeploy, hostPort, contPort
                    imageValidation().call()
                    dockerDeploy('prd', '8132', '8132').call()
                }       
            }
        }       
    }
}


//
def buildApp(){
    return {
        echo "Building the ${env.APPLICATION_NAME} Application"
        sh 'mvn clean package -DskipTests=true'
        // After building if you face any issue with mvn installation? --> There is issue with the permissions (chown -R <kommuamar1133>:<kommuamar1133> <apache maven) of maven in /opt/apcahe_maven
    }
}

//
def sonar(){
    return {
        echo "Starting Sonar Scan"
        withSonarQubeEnv('SonarQube'){  // The name you saved in system under manage jenkins
            sh """
            mvn sonar:sonar \
                -Dsonar.projectkey=i27-eureka \
                -Dsonar.host.url=${env.SONAR_URL} \
                -Dsonar.login=${SONAR_TOKEN}
            """
        }  
        timeout (time: 2, unit: 'MINUTES'){
            waitForQualityGate abortPipeline: true
        }
    }
}

// Method for docker build & push
def dockerBuildAndPush(){
    return {
        // Existing artifact format: i27-eureka-0.0.1-SNAPSHOT.jar
        // My Destination artifact format: i27-eureka-buildnumber-branchname.jar
        echo "My JAR file SOURCE: i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING}"
        echo "My JAR Destination: i27-${env.APPLICATION_NAME}-${BUILD_NUMBER}-${BRANCH_NAME}.${env.POM_PACKAGING}"
        sh """
            echo "***********************Building Docker Image*************************"
            pwd
            ls -la
            cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING}  ./.cicd
            ls -la ./.cicd
            docker build --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd
            # docker build -t imagename dockerfilepath
            echo "***********************Login to Docker Registry*******************************"
            # docker login -u username -p password
            docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}
            # docker push image_name
            docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}
        """
    }
}


def imageValidation(){
    return {
        println("Attempting to Pull the Docker Image")
        try {
            sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
            println("Image Pulled Succesfully!!!!")
        }
        catch(Exception e){
            println("OPPS!, the docker image with this tag is not available, So creating the Image")
            buildApp().call()
            sonar().call()
            dockerBuildAndPush().call()
        }
    }
}

// Method for deploying containers in diff envs
def dockerDeploy(envDeploy, hostPort, contPort){
    return {
        echo "Deploying to $envDeploy Environment"
        withCredentials([usernamePassword(credentialsId: 'navya_ssh_dockerserver_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
            script {
                //sshpass -p password ssh -o StrictHostKeyChecking=no username@dockerserver_ip
                sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip \"docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}\""
                try {
                    // Stop container
                    sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip docker stop ${env.APPLICATION_NAME}-$envDeploy"
                    // Remove Container
                    sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip docker rm ${env.APPLICATION_NAME}-$envDeploy"
                }
                catch(err) {
                    echo "Error Caught: $err"
                }
                //Create container
                sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip docker run -dit --name ${env.APPLICATION_NAME}-$envDeploy -p $hostPort:$contPort ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
            }
        }  
    }
}






// container port = 8761

// dev hp = 5761
// tst hp = 6761
// stg hp = 7761
// prod hp = 8761































//We need to connect to the dockerserver through the jenkinsslave using below command:
//withCredentials([usernameColonPassword(credentialsId: 'mylogin', variable: 'USERPASS')]) {
  // sshpass -p password ssh -o StrictHostKeyChecking=no username@dockerserver_ip



// usernameVariable : String
// Name of an environment variable to be set to the username during the build.
// passwordVariable : String
// Name of an environment variable to be set to the password during the build.
// credentialsId : String
// Credentials of an appropriate type to be set to the variable.