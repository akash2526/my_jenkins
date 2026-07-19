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
        IMAGE_NAME = "vmimage"
        CONTAINER_NAME = "vmimage"
    }

    stages {

        stage('Load Configuration') {

            steps {

                script {

                    env.IMAGE_TAG = "${BUILD_NUMBER}"

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

        stage('Debug Variables') {

            steps {

                sh '''
                echo "==============================="
                echo "BUILD_NUMBER=$BUILD_NUMBER"
                echo "IMAGE_TAG=$IMAGE_TAG"
                echo "REGION=$REGION"

                echo "DEV_PROJECT=$DEV_PROJECT"
                echo "DEV_IMAGE=$DEV_IMAGE"
                echo "DEV_VM=$DEV_VM"
                echo "DEV_USER=$DEV_USER"

                echo "UAT_PROJECT=$UAT_PROJECT"
                echo "UAT_IMAGE=$UAT_IMAGE"
                echo "==============================="
                '''

            }

        }

        stage('Clone Repository') {

            steps {

                git(
                    branch: 'main',
                    credentialsId: 'Github-access',
                    url: 'https://github.com/kamalateck/vm-pipeline.git'
                )

            }

        }

        stage('Build Docker Image') {

            steps {

                sh """

                echo "Current Workspace"
                pwd

                ls -la

                echo "Building Docker Image..."

                DOCKER_BUILDKIT=0 docker build \
                -t ${DEV_IMAGE}:${IMAGE_TAG} .

                echo ""

                echo "Docker Images"

                docker images | grep vmimage || true

                echo ""

                docker image inspect ${DEV_IMAGE}:${IMAGE_TAG}

                """

            }

        }

        stage('Authenticate Dev GCP') {

            steps {

                withCredentials([

                    file(
                        credentialsId: 'service-account-dev',
                        variable: 'GOOGLE_APPLICATION_CREDENTIALS'
                    )

                ]) {

                    sh """

                    echo "Activating Service Account..."

                    gcloud auth activate-service-account \
                    --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                    gcloud config set project ${DEV_PROJECT}

                    gcloud auth list

                    gcloud auth configure-docker \
                    ${REGION}-docker.pkg.dev \
                    --quiet

                    """

                }

            }

        }

        stage('Push Image To Artifact Registry') {

            steps {

                sh """

                echo "Pushing Docker Image..."

                docker push ${DEV_IMAGE}:${IMAGE_TAG}

                """

            }

        }

        stage('Verify Artifact Registry') {

            steps {

                sh """

                echo "Checking image in Artifact Registry..."

                gcloud artifacts docker images list \
                ${DEV_IMAGE} \
                --include-tags

                """

            }

        }

stage('Deploy Dev VM') {

    steps {

        withCredentials([
            sshUserPrivateKey(
                credentialsId: 'VM-dev-ssh-key',
                keyFileVariable: 'SSH_KEY'
            )
        ]) {

            sh """
chmod 600 "\$SSH_KEY"

ssh -o StrictHostKeyChecking=no \
-i "\$SSH_KEY" \
${DEV_USER}@${DEV_VM} <<EOF

set -e

echo "Docker Version"
docker --version

echo "Configuring Docker Authentication"
gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet || true

echo "Stopping old container"
docker stop ${CONTAINER_NAME} || true
docker rm ${CONTAINER_NAME} || true

echo "Pulling Image"
docker pull ${DEV_IMAGE}:${IMAGE_TAG}

echo "Running Container"
docker run -d \
  --restart unless-stopped \
  -p 8000:8000 \
  --name ${CONTAINER_NAME} \
  ${DEV_IMAGE}:${IMAGE_TAG}

sleep 5

docker ps

curl -I http://localhost:8000 || true

exit
EOF
"""

        }

    }

}

        stage('Manual Approval For UAT') {

            steps {

                input(
                    message: 'Dev Deployment Successful. Deploy to UAT?',
                    ok: 'Deploy'
                )

            }

        }

        stage('Authenticate UAT GCP') {

            steps {

                withCredentials([

                    file(
                        credentialsId: 'service-account-UAT',
                        variable: 'GOOGLE_APPLICATION_CREDENTIALS'
                    )

                ]) {

                    sh """

                    gcloud auth activate-service-account \
                    --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                    gcloud config set project ${UAT_PROJECT}

                    gcloud auth configure-docker \
                    ${REGION}-docker.pkg.dev \
                    --quiet

                    """

                }

            }

        }

        stage('Promote Image To UAT') {

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
                        credentialsId: 'VM-UAT-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )

                ]) {

                    sh """

                    chmod 600 \$SSH_KEY

                    ssh \
                    -o StrictHostKeyChecking=no \
                    -i \$SSH_KEY \
                    ${UAT_USER}@${UAT_VM} << 'EOF'

                    set -e

                    gcloud auth configure-docker \
                    ${REGION}-docker.pkg.dev \
                    --quiet || true

                    docker stop ${CONTAINER_NAME} || true

                    docker rm ${CONTAINER_NAME} || true

                    docker pull ${UAT_IMAGE}:${IMAGE_TAG} || exit 1

                    docker run -d \
                    --restart unless-stopped \
                    -p 8000:8000 \
                    --name ${CONTAINER_NAME} \
                    ${UAT_IMAGE}:${IMAGE_TAG}

                    sleep 10

                    docker ps

                    curl http://localhost:8000 || true

                    EOF

                    """

                }

            }

        }

    }

    post {

        always {

            echo "Cleaning Workspace..."

            cleanWs(
                deleteDirs: true,
                disableDeferredWipeout: true
            )

        }

        success {

            echo "Pipeline Completed Successfully."

        }

        failure {

            echo "Pipeline Failed."

        }

    }

}