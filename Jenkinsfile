pipeline {
    agent any

    tools {
         terraform 'tf'
    }

    environment {
        IMAGE_NAME = "refiakayaalp/java-app:java-maven-${BUILD_NUMBER}"
    }

    stages {
        stage('Build app') {
            steps {
                script {
                    echo "building the docker image"
                    withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {            

                        sh 'docker build -t ${IMAGE_NAME} .'
                        sh 'echo $PASSWORD | docker login -u $USER --password-stdin '
                        sh 'docker push ${IMAGE_NAME}'
                    }
                } 
            }
        }



        stage('Provision Server') {
            steps {
                script {
                    echo "provisionin server"
                    dir('Terraform') {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',credentialsId: "AWS-ID",accessKeyVariable: 'AWS_ACCESS_KEY_ID',secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                            sh "terraform init"
                            sh "terraform apply --auto-approve"
                            EC2_PUBLIC_IP = sh(script: "terraform output ec2_public_ip",returnStdout: true).trim()
                        }
                    }
                } 
            }
        }


        stage('Deploy') {
            enviroment {
                DOCKER_CREDS = credentials('docker-hub')
            }
            
            steps {
                script {
                    echo "provisionin server"
                    echo "waiting for ec2 server to initialize"
                    sleep(time: 90, unit: "SECONDS")
                    echo "deploying docker image to ec2"
                    echo "${EC2_PUBLIC_IP}"
                    def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
                    def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

                    sshagent(['ssh-key']) {
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
   
                    }
                }   
            } 
        }
    }
}

