pipeline {

    agent any

    environment {

        IMAGE_NAME = 'vmimage'
        CONTAINER_NAME = 'vmimage'
        REGION = 'europe-west2'

    }

    stages {

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
                -t ${DEV_IMAGE_PATH}:${BUILD_NUMBER} .

                """

            }
        }

        stage('Authenticate Dev GCP') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'gcp-service-account-dev',
                        variable: 'GOOGLE_APPLICATION_CREDENTIALS'
                    )
                ]) {

                    sh """

                    gcloud auth activate-service-account \
                    --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                    gcloud config set project ${DEV_PROJECT_ID}

                    gcloud auth configure-docker \
                    ${REGION}-docker.pkg.dev --quiet

                    """

                }

            }
        }

        stage('Push Image To Dev Artifact Registry') {

            steps {

                sh """

                docker push ${DEV_IMAGE_PATH}:${BUILD_NUMBER}

                """

            }
        }

        stage('Deploy To Dev VM') {

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
                    ${DEV_VM_USER}@${DEV_VM_IP} << EOF


                    echo "Configuring Docker authentication"

                    gcloud auth configure-docker \
                    ${REGION}-docker.pkg.dev --quiet


                    echo "Stopping old Dev container"

                    docker stop ${CONTAINER_NAME} || true

                    docker rm ${CONTAINER_NAME} || true


                    echo "Pulling Dev image"

                    docker pull ${DEV_IMAGE_PATH}:${BUILD_NUMBER}


                    echo "Starting Dev container"

                    docker run -d \
                    --restart always \
                    -p 8000:8000 \
                    --name ${CONTAINER_NAME} \
                    ${DEV_IMAGE_PATH}:${BUILD_NUMBER}


EOF

                    """

                }

            }

        }


        stage('Manual Approval For UAT') {

            steps {

                input(
                    message: 'Dev deployment completed. Approve deployment to UAT?',
                    ok: 'Deploy To UAT'
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
                    --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                    gcloud config set project ${UAT_PROJECT_ID}

                    gcloud auth configure-docker \
                    ${REGION}-docker.pkg.dev --quiet

                    """

                }

            }

        }


        stage('Tag And Push Image To UAT Artifact Registry') {

            steps {

                sh """

                docker tag \
                ${DEV_IMAGE_PATH}:${BUILD_NUMBER} \
                ${UAT_IMAGE_PATH}:${BUILD_NUMBER}


                docker push \
                ${UAT_IMAGE_PATH}:${BUILD_NUMBER}

                """

            }

        }


        stage('Deploy To UAT VM') {

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
                    ${UAT_VM_USER}@${UAT_VM_IP} << EOF


                    echo "Configuring Docker authentication"

                    gcloud auth configure-docker \
                    ${REGION}-docker.pkg.dev --quiet


                    echo "Stopping old UAT container"

                    docker stop ${CONTAINER_NAME} || true

                    docker rm ${CONTAINER_NAME} || true


                    echo "Pulling UAT image"

                    docker pull ${UAT_IMAGE_PATH}:${BUILD_NUMBER}


                    echo "Starting UAT container"

                    docker run -d \
                    --restart always \
                    -p 8000:8000 \
                    --name ${CONTAINER_NAME} \
                    ${UAT_IMAGE_PATH}:${BUILD_NUMBER}


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
