pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'  // Docker Hub
        IMAGE_NAME = 'jeromeevangelista/simple-webapp'  // Image name on Docker Hub
        DOCKER_CREDENTIALS = 'docker-creds'  // Jenkins credentials ID for Docker login

        // MinIO Configuration (internal DNS)
        MINIO_URL = 'http://minio.minio:9000'  // Use internal DNS
        MINIO_BUCKET = 'simple-webapp'  // MinIO bucket name
        
        merged = "${params.merged}"
        state = "${params.state}"
        branch = "${params.branch}"
        action = "${params.action}"
    }

    stages {
        stage('Check PR Conditions') {
            steps {
                script {
                    // Check if the conditions are met

                    echo "$merged"
                    echo "$state"
                    echo "$branch"
                    echo "$action"
                    if (merged == 'true' && state == 'closed' && branch == 'main' && action == 'closed') {
                        echo "PR was merged into 'main' and is closed. Proceeding with build."
                    } else {
                        echo "Conditions not met. Skipping build."
                        currentBuild.result = 'ABORTED'
                        error('Stopping early…')
                        return // Stop the pipeline if conditions are not met
                    }
                }
            }
        }
        
        stage('Checkout') {
            steps {
                script {
                    echo 'Checkout the code from the GitHub repository'
                    checkout scm
                }
            }
        }

        stage('Get Current Version') {
            steps {
                script {
                    echo 'Check if version.txt exists and read the version from it'
                    if (fileExists('version.txt')) {
                        VERSION = readFile('version.txt').trim()
                    } else {
                        VERSION = 'latest'
                    }

                    echo "Current version: ${VERSION}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Build the Docker image with the new version tag'
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${VERSION}", "webapp/")
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                script {
                    echo 'Login to Docker Hub'
                    withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh '''
                            docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}
                        '''
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo 'Push the Docker image'
                    docker.withRegistry('', DOCKER_CREDENTIALS) {
                        docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${VERSION}").push()
                    }
                }
            }
        }

        stage('Manipulate Files') {
            steps {
                script {
                    echo 'Extract background color'
                    def backgroundColor = sh(script: "grep -o 'background-color:[^\\\"]*' webapp/templates/index.html | cut -d ':' -f 2 | tr -d ' '", returnStdout: true).trim()
                    echo "Extracted background color: ${backgroundColor}"

                    echo 'Read and modify the template file'
                    def templateFile = readFile('manifests/v1/template-webapp')
                    def templateFile2 = readFile('manifests/v1/template-canary')
                    def updatedFile = templateFile.replace("{{color}}", backgroundColor).replace("{{tag}}", VERSION)
                    def updatedFile2 = templateFile2.replace("{{color}}", backgroundColor).replace("{{tag}}", VERSION)

                    echo 'Save the updated file as canary.yml and production.yml'
                    writeFile file: 'manifests/v1/canary.yml', text: updatedFile2
                    writeFile file: 'manifests/v1/production.yml', text: updatedFile

                    echo "Replaced placeholders in template and saved to canary.yml and production.yml"
                }
            }
        }

        stage('Upload Files to MinIO') {
            steps {
                script {

                    echo 'Configuring MinIO client (mc) with the internal DNS for minio service'

                    withCredentials([usernamePassword(credentialsId: 'minio-creds', usernameVariable: 'MINIO_ACCESS_KEY', passwordVariable: 'MINIO_SECRET_KEY')]) {
                        sh """
                            mc alias set myminio ${MINIO_URL} ${MINIO_ACCESS_KEY} ${MINIO_SECRET_KEY}
                        """

                        echo 'Upload production.yml and canary.yml to MinIO bucket'
                        sh """
                            mc cp manifests/v1/production.yml myminio/${MINIO_BUCKET}/${VERSION}/production.yml
                            mc cp manifests/v1/canary.yml myminio/${MINIO_BUCKET}/${VERSION}/canary.yml
                        """

                        echo 'Files uploaded to MinIO successfully'
                    }
                }
            }
        }

        stage('Send Trigger to Spinnaker') {
            steps {
                script {
                    sh "curl spin-gate.spinnaker:8084/webhooks/webhook/simple-webapp-deploy -X POST -H \"content-type: application/json\" -d '{ \"parameters\": { \"version\": \"${VERSION}\" } }'"
                }
            }
        }

        stage('Clean Up') {
            steps {
                script {
                    echo 'Remove local Docker images'
                    sh "docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:${VERSION}"
                }
            }
        }
    }

    post {
        always {
            echo 'Clean up the workspace after the build'
            cleanWs()
        }
        success {
            echo "Docker image successfully built and pushed with version tag ${VERSION}"
        }
        failure {
            echo "Build failed."
        }
    }
}
