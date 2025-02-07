pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_CREDENTIALS_ID = 'docker-hub'
        DOCKER_IMAGE = 'blackopsgun/pet-clinic'
        DOCKER_TAG = "build-${env.BUILD_NUMBER}"
        GIT_REPO = "HarshwardhanBaghel/G6-Project"
        DEPLOYMENT_FILE = "deployment/deployment.yml"
    }

    stages {
        stage('Checkout Code') {
            steps {
                cleanWs()
                git branch: 'main', credentialsId: 'github-account', url: "https://github.com/${GIT_REPO}.git"
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn clean package'
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
                dependencyCheck additionalArguments: '--scan ./ --format HTML --out .', odcInstallation: 'DP-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.html'
            }
        }

            stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker logout
                    """
                }
            }
        }



        stage('Update Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config --global user.email "gudduharsh29@gmail.com"
                        git config --global user.name "HarshwardhanBaghel"
                        
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${DOCKER_TAG}|" ${DEPLOYMENT_FILE}

                        git add ${DEPLOYMENT_FILE}
                        git commit -m "Update deployment image to ${DOCKER_TAG}" || echo "No changes to commit"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_REPO}.git HEAD:main
                    '''
                }
            }
        }
    }

    post {
        always {
            mail to: 'gudduharsh29@gmail.com',
                 subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.result}",
                 body: "Check the build details at ${env.BUILD_URL}"
        }
    }
}
