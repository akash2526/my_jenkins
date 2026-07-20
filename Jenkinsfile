pipeline {

    agent any


    parameters {

        choice(
            name: 'DEPLOY_TARGET',
            choices: ['dev'],
            description: 'Start deployment from Dev environment'
        )

    }


    environment {

        IMAGE_NAME = 'vmimage'

        CONTAINER_NAME = 'vmimage'

    }


    stages {


        stage('Load Configuration') {

            steps {

                script {


                    // Current Jenkins build number becomes image tag

                    env.IMAGE_TAG = "${BUILD_NUMBER}"


                    // Previous image for rollback

                    env.PREVIOUS_IMAGE_TAG = "${BUILD_NUMBER.toInteger() - 1}"



                    env.DEV_PROJECT = DEV_PROJECT_ID

                    env.DEV_IMAGE = DEV_IMAGE_PATH

                    env.DEV_VM = DEV_VM_IP

                    env.DEV_USER = DEV_VM_USER



                    env.UAT_PROJECT = UAT_PROJECT_ID

                    env.UAT_IMAGE = UAT_IMAGE_PATH

                    env.UAT_VM = UAT_VM_IP

                    env.UAT_USER = UAT_VM_USER


                }

            }

        }



        stage('Clone Repository') {


            steps {


                git(

                    branch: 'main',

                    url: 'https://github.com/kamalateck/vm-pipeline.git'

                )


            }

        }




        stage('Build Docker Image') {


            steps {


                sh """

                docker build \

                -t ${DEV_IMAGE}:${IMAGE_TAG} .


                """

            }

        }




        stage('Authenticate DEV GCP') {


            steps {


                withCredentials([

                    file(

                        credentialsId: 'gcp-service-account-dev',

                        variable: 'GOOGLE_APPLICATION_CREDENTIALS'

                    )

                ]) {


                    sh """


                    gcloud auth activate-service-account \

                    --key-file=$GOOGLE_APPLICATION_CREDENTIALS



                    gcloud config set project ${DEV_PROJECT}



                    gcloud auth configure-docker \

                    ${REGION}-docker.pkg.dev \

                    --quiet


                    """

                }

            }

        }





        stage('Push Image To DEV Artifact Registry') {


            steps {


                sh """

                docker push ${DEV_IMAGE}:${IMAGE_TAG}

                """

            }

        }





        stage('Deploy DEV VM') {


            steps {


                withCredentials([


                    sshUserPrivateKey(

                        credentialsId: 'vm-ssh-key',

                        keyFileVariable: 'SSH_KEY'

                    )

                ]) {


                    sh """


                    chmod 600 \$SSH_KEY



                    ssh \

                    -o StrictHostKeyChecking=no \

                    -i \$SSH_KEY \

                    ${DEV_USER}@${DEV_VM} << EOF



                    echo "Stopping old container"



                    docker stop ${CONTAINER_NAME} || true


                    docker rm ${CONTAINER_NAME} || true




                    echo "Pulling new image"



                    docker pull ${DEV_IMAGE}:${IMAGE_TAG}




                    echo "Starting new container"



                    docker run -d \

                    --restart always \

                    -p 8000:8000 \

                    --name ${CONTAINER_NAME} \

                    ${DEV_IMAGE}:${IMAGE_TAG}



EOF


                    """

                }

            }

        }







        stage('Health Check DEV') {


            steps {


                script {


                    echo "Checking DEV application"



                    def health = sh(

                        script: """

                        curl -f http://${DEV_VM}:8000

                        """,

                        returnStatus: true

                    )



                    if(health != 0) {


                        error("DEV health check failed")


                    }


                    echo "DEV application is healthy"


                }

            }

        }






        stage('Rollback DEV') {


            when {


                expression {


                    currentBuild.currentResult == 'FAILURE'


                }

            }



            steps {


                echo "Rollback started"



                withCredentials([


                    sshUserPrivateKey(

                        credentialsId: 'vm-ssh-key',

                        keyFileVariable: 'SSH_KEY'

                    )

                ]) {


                    sh """


                    chmod 600 \$SSH_KEY



                    ssh \

                    -o StrictHostKeyChecking=no \

                    -i \$SSH_KEY \

                    ${DEV_USER}@${DEV_VM} << EOF




                    echo "Stopping failed version"



                    docker stop ${CONTAINER_NAME} || true


                    docker rm ${CONTAINER_NAME} || true





                    echo "Pulling previous image"



                    docker pull ${DEV_IMAGE}:${PREVIOUS_IMAGE_TAG}





                    echo "Starting rollback version"



                    docker run -d \

                    --restart always \

                    -p 8000:8000 \

                    --name ${CONTAINER_NAME} \

                    ${DEV_IMAGE}:${PREVIOUS_IMAGE_TAG}



EOF


                    """

                }

            }

        }







        stage('Manual Approval For UAT') {


            steps {


                input(

                    message: 'DEV deployment completed. Deploy to UAT?',

                    ok: 'Deploy UAT'

                )


            }

        }







        stage('Authenticate UAT GCP') {


            steps {


                withCredentials([


                    file(

                        credentialsId: 'gcp-service-account-uat',

                        variable: 'GOOGLE_APPLICATION_CREDENTIALS'

                    )

                ]) {


                    sh """


                    gcloud auth activate-service-account \

                    --key-file=$GOOGLE_APPLICATION_CREDENTIALS



                    gcloud config set project ${UAT_PROJECT}



                    gcloud auth configure-docker \

                    ${REGION}-docker.pkg.dev \

                    --quiet


                    """

                }

            }

        }







        stage('Promote Image To UAT Registry') {


            steps {


                sh """


                docker tag \

                ${DEV_IMAGE}:${IMAGE_TAG} \

                ${UAT_IMAGE}:${IMAGE_TAG}




                docker push \

                ${UAT_IMAGE}:${IMAGE_TAG}



                """

            }

        }







        stage('Deploy UAT VM') {


            steps {


                withCredentials([


                    sshUserPrivateKey(

                        credentialsId: 'vm-uat-ssh-key',

                        keyFileVariable: 'SSH_KEY'

                    )

                ]) {



                    sh """


                    chmod 600 \$SSH_KEY




                    ssh \

                    -o StrictHostKeyChecking=no \

                    -i \$SSH_KEY \

                    ${UAT_USER}@${UAT_VM} << EOF



                    docker stop ${CONTAINER_NAME} || true


                    docker rm ${CONTAINER_NAME} || true




                    docker pull ${UAT_IMAGE}:${IMAGE_TAG}




                    docker run -d \

                    --restart always \

                    -p 8000:8000 \

                    --name ${CONTAINER_NAME} \

                    ${UAT_IMAGE}:${IMAGE_TAG}



EOF


                    """

                }

            }

        }







        stage('Cleanup Workspace') {


            steps {


                cleanWs()


            }

        }


    }


}
