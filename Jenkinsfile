pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = '505c5650-5acc-4d9d-b682-0f963d56b65d'
        GIT_CREDENTIALS_ID = '36e7e1ba-9b70-42dd-b9f0-b3d473dd3d0f	'
        HARBOR_URL = 'selfhosted.harbor.com:443'
        HARBOR_REPOSITORY = 'library/finexo'
        DOCKER_IMAGE_NAME = 'finexo'
        DOCKER_IMAGE_TAG = 'latest'
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Clone Repository') {
            steps {
                sh " git config --global http.sslVerify false "
                git(
                    url: 'https://mygitlabinstance.com/root/my-cicd.git',
                    branch: 'main',
                    credentialsId: "${GIT_CREDENTIALS_ID}",
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    try {
                        docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}")
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }

        stage('Tag Docker Image') {
            steps {
                script {
                    try {
                        sh "docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ${HARBOR_URL}/${HARBOR_REPOSITORY}:${DOCKER_IMAGE_TAG}"
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }

        stage('Push Docker Image to Harbor') {
            steps {
                script {
                    try {
                        docker.withRegistry("https://${HARBOR_URL}", "${DOCKER_CREDENTIALS_ID}") {
                            sh "docker push ${HARBOR_URL}/${HARBOR_REPOSITORY}:${DOCKER_IMAGE_TAG}"
                        }
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }

        stage('Clone Kubernetes Deployment Git Repository') {
            steps {
                git branch: 'main', credentialsId: "${GIT_CREDENTIALS_ID}", url: 'https://mygitlabinstance.com/root/deployment-repo.git'
            }
        }
        
        stage('Update Image Version') {
            steps {
                script {
                    echo "Before update"
                    sh 'cat app.yaml'
        
                    // Pass IMAGE_DIGEST to the shell script
                    def digest = sh(script: 'docker inspect --format="{{index .RepoDigests 0}}" ${HARBOR_URL}/${HARBOR_REPOSITORY}:${DOCKER_IMAGE_TAG}', returnStdout: true).trim()
                    echo "Image Digest: ${digest}"
                    IMAGE_DIGEST = digest // Store complete digest
                    sh """
                        #!/bin/bash -xe
                        perl -i -pe 's|image: selfhosted.*|image: ${IMAGE_DIGEST}|' app.yaml
                    """
        
                    // Confirm changes
                    echo "After Update"
                    sh 'cat app.yaml'
        
                    // Check for changes and commit
                    def status = sh(script: 'git status --porcelain', returnStdout: true).trim()
                    if (status) {
                        sh """
                            git add app.yaml
                            git commit -m "Update image version to digest ${IMAGE_DIGEST}"
                        """
                    } else {
                        echo 'No changes to commit'
                    }
                }
            }
        }


        stage('Push to Git Repository') {   
            steps {
                withCredentials([gitUsernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", gitToolName: 'Default')]) {
                    sh 'git push -u origin main'
                }
            }
        }

        stage('Clean Up') {
        steps {
            script {
                // List and remove Docker images
                sh '''
                    # List all image IDs matching the filter
                    image_ids=$(docker images --filter=reference='*finexo:*' -q)
                    
                    # Check if there are any images to remove
                    if [ -n "$image_ids" ]; then
                        # Remove the images
                        docker rmi $image_ids -f
                    else
                        echo "No images to remove."
                    fi
                    
                    # Clean up workspace
                    rm -rf ${WORKSPACE}/*
                '''
                }
            }
        }
    }

    post {
        success {
            echo 'Docker image built, pushed, and app.yaml updated successfully!'
        }
        failure {
            echo 'Build failed.'
        }
    }
}
