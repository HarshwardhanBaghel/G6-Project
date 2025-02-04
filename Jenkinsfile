pipeline {
    agent any

    tools {
        jdk 'jdk11'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_CREDENTIALS_ID = 'docker-hub'
        DOCKER_IMAGE = 'blackopsgun/pet-clinic'
        DOCKER_TAG = "build-${env.BUILD_NUMBER}"  // Unique build tag
        DEPLOYMENT_FILE = "deployment/deployment.yml"
        GIT_USER_NAME = "HarshwardhanBaghel"
        GIT_REPO_NAME = "G6-Project"
    }

    stages {
        stage('Checkout Code') {
            steps {
                cleanWs()  // Ensure clean workspace for each build
                git branch: 'main', url: "https://github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git"
            }
        }

        stage('Compile Code') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Static Code Analysis') {
            steps {
                withSonarQubeEnv('Sonar-server') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=Petclinic \
                        -Dsonar.projectName=Petclinic \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Dependency Vulnerability Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML --out .', odcInstallation: 'DP-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_CREDENTIALS_ID) {
                        def appImage = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                        appImage.push()
                    }
                }
            }
        }

        stage('Update Deployment File') {
    steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
            sh '''
                git config --global user.email "gudduharsh29@gmail.com"
                git config --global user.name "HarshwardhanBaghel"
                
                # Stash any local changes before pulling
                git stash save "temporary changes"
                
                # Pull the latest changes from the main branch
                git pull --rebase origin main
                
                # Ensure the placeholder exists in the deployment.yml file
                if ! grep -q 'replaceImageTag' deployment/deployment.yml; then
                    echo "replaceImageTag placeholder not found. Adding it back."
                    sed -i 's/image: .*/image: blackopsgun\/pet-clinic:replaceImageTag/' deployment/deployment.yml
                fi
                
                # Update image tag in deployment.yml with the build number
                sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" deployment/deployment.yml
                
                # Check if there are changes to commit
                git diff --quiet || (git add deployment/deployment.yml && git commit -m "Update deployment image to version ${BUILD_NUMBER}")
                
                # Push the changes to the remote repository
                git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                
                # Apply stashed changes (if any)
                git stash pop
            '''
        }
    }
}



    }

    post {
        success {
            mail to: 'gudduharsh29@gmail.com',
                 subject: "SUCCESS: Build ${env.BUILD_NUMBER}",
                 body: "The build was successful. Docker image: ${DOCKER_IMAGE}:${DOCKER_TAG}. Check the details at ${env.BUILD_URL}"
        }
        failure {
            mail to: 'gudduharsh29@gmail.com',
                 subject: "FAILURE: Build ${env.BUILD_NUMBER}",
                 body: "The build failed. Check the details at ${env.BUILD_URL}"
        }
    }
}
