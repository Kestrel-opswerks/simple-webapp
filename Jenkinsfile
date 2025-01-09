pipeline {
    agent any

    triggers {
        // Trigger the pipeline on push to any branch or a pull request
        githubPush()  // This triggers the build on any push to GitHub
    }

    stages {
        stage('Test') {
            steps {
                script {
                    // Echo "test" message
                    echo "test"
                }
            }
        }
    }
}
