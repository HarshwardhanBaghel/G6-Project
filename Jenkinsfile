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
        LATEST_TAG = "latest"
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
                        sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:${LATEST_TAG}"
                        appImage.push()
                        sh "docker push ${DOCKER_IMAGE}:${LATEST_TAG}"
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
                        
                        # Pull the latest changes
                        git pull --rebase origin main
                        
                        # Update image tag in deployment.yml
                        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" deployment/deployment.yml
                        
                        # Check if there are changes
                        git diff --quiet || (git add ${DEPLOYMENT_FILE} && git commit -m "Update deployment image to version ${BUILD_NUMBER}")
                        
                        # Push the changes
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
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
