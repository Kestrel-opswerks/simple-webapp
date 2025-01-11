pipeline {
    agent any

    environment {
        // Define your Docker registry and image name
        DOCKER_REGISTRY = 'docker.io'  // Docker Hub
        IMAGE_NAME = 'jeromeevangelista/simple-webapp'  // Your image name on Docker Hub
        DOCKER_CREDENTIALS = 'docker-creds'  // Jenkins credentials ID for Docker login
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    // Checkout the code from the GitHub repository
                    checkout scm
                }
            }
        }

        stage('Get Current Version') {
            steps {
                script {
                    // Check if version.txt exists and read the version from it
                    if (fileExists('version.txt')) {
                        VERSION = readFile('version.txt').trim()
                    } else {
                        // Default version if version.txt doesn't exist
                        VERSION = '0.0.1'
                    }

                    // Increment the version (assumes version format: x.y.z)
                    def versionParts = VERSION.tokenize('.')
                    def patch = versionParts[2].toInteger() + 1
                    VERSION = "${versionParts[0]}.${versionParts[1]}.${patch}"

                    // Save the incremented version back to version.txt
                    writeFile file: 'version.txt', text: VERSION

                    echo "Current version: ${VERSION}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image with the new version tag
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${VERSION}", "webapp/")
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                script {
                    // Login to Docker registry (Docker Hub)
                    docker.withRegistry('', DOCKER_CREDENTIALS) {
                        // This will use the credentials stored in Jenkins
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    // Push the Docker image to Docker Hub with the version tag
                    docker.withRegistry('', DOCKER_CREDENTIALS) {
                        docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${VERSION}").push()
                    }
                }
            }
        }

        stage('Manipulate Files') {
            steps {
                script {
                    // Extract background color from webapp/templates/index.html
                    def backgroundColor = sh(script: "grep -o 'background-color:[^;']*' webapp/templates/index.html | cut -d ':' -f 2 | tr -d ' '", returnStdout: true).trim()
                    echo "Extracted background color: ${backgroundColor}"

                    // Read and modify the template file
                    def templateFile = readFile('manifests/v1/template-webapp')
                    def updatedFile = templateFile.replace("{{color}}", backgroundColor).replace("{{tag}}", VERSION)

                    // Save the updated file as webapp-canary.yml and webapp.yml
                    writeFile file: 'webapp-canary.yml', text: updatedFile
                    writeFile file: 'webapp.yml', text: updatedFile

                    echo "Replaced placeholders in template and saved to webapp-canary.yml and webapp.yml"
                }
            }
        }

        stage('Commit and Push to Test Branch') {
            steps {
                script {
                    // Checkout to the test branch
                    sh 'git checkout -b test || git checkout test'

                    // Add the new files to Git
                    sh 'git add webapp-canary.yml webapp.yml version.txt'

                    // Commit the changes with a message
                    sh 'git commit -m "Add webapp-canary.yml, webapp.yml and version.txt files with updated version and background color"'

                    // Push the changes to the test branch
                    withCredentials([usernamePassword(credentialsId: 'github-repo', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                            git config user.name '${GIT_USER}'
                            git config user.email 'franz.lopez@academy.opswerks.com'
                            git push https://${GIT_USER}:${GIT_PASSWORD}@github.com/your-org/your-repo.git test
                        """
                    }

                    echo "Files committed and pushed to the test branch"
                }
            }
        }

        stage('Clean Up') {
            steps {
                script {
                    // Remove locally stored Docker images to free up space
                    sh "docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:${VERSION}"
                }
            }
        }
    }

    post {
        always {
            // Clean up the workspace after the build, no matter if it succeeds or fails
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
