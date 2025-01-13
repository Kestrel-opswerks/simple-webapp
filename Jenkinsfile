pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'  // Docker Hub
        IMAGE_NAME = 'jeromeevangelista/simple-webapp'  // Image name on Docker Hub
        DOCKER_CREDENTIALS = 'docker-creds'  // Jenkins credentials ID for Docker login

        // MinIO Configuration (internal DNS)
        MINIO_URL = 'http://minio.minio:9000'  // Use internal DNS
        MINIO_BUCKET = 'simple-webapp'  // MinIO bucket name
    }

    stages {
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
                        VERSION = '1.0.0'
                    }

                    echo 'Increment the version'
                    def versionParts = VERSION.tokenize('.')
                    def patch = versionParts[2].toInteger() + 1
                    VERSION = "${versionParts[0]}.${versionParts[1]}.${patch}"

                    echo 'Save version.txt'
                    writeFile file: 'version.txt', text: VERSION

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
                    def updatedFile = templateFile.replace("{{color}}", backgroundColor).replace("{{tag}}", VERSION)

                    echo 'Save the updated file as webapp-canary.yml and webapp.yml'
                    writeFile file: 'manifests/v1/canary.yml', text: updatedFile
                    writeFile file: 'manifests/v1/production.yml', text: updatedFile

                    echo "Replaced placeholders in template and saved to webapp-canary.yml and webapp.yml"
                }
            }
        }

        stage('Commit and Push to Test Branch') {
            steps {
                script {
                    echo 'Checkout to the main branch'
                    sh 'git checkout -b main || git checkout main'
                    echo 'Add the new files to Git'
                    sh 'git add manifests/v1/canary.yml manifests/v1/production.yml version.txt'

                    withCredentials([usernamePassword(credentialsId: 'github-repo', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            git config user.name '${GIT_USER}'
                            git config user.email 'franz.lopez@academy.opswerks.com'
                            git commit -m "Add canary.yml, production.yml and version.txt files with updated version and background color"
                        '''
                    }

                    echo 'Commit changes to local'

                    withCredentials([usernamePassword(credentialsId: 'github-repo', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            git config user.name '${GIT_USER}'
                            git config user.email 'franz.lopez@academy.opswerks.com'
                            git push https://${GIT_USER}:${GIT_PASSWORD}@github.com/Kestrel-opswerks/simple-webapp.git main
                        '''
                    }

                    echo "Files committed and pushed to the remote main branch"
                }
            }
        }

        stage('Upload Files to MinIO') {
            steps {
                script {

                    echo 'Configuring MinIO client (mc) with the internal DNS for minio service'

                    // Fetch MinIO credentials from Jenkins and configure mc
                    withCredentials([usernamePassword(credentialsId: 'minio-creds', usernameVariable: 'MINIO_ACCESS_KEY', passwordVariable: 'MINIO_SECRET_KEY')]) {
                        // Configure mc to access MinIO
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
