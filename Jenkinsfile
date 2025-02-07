pipeline {
    agent any

    tools {
        jdk 'jdk11'
        maven 'maven3'
    }

    environment {
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

        stage('Static & Dependency Analysis') {
            steps {
                withSonarQubeEnv('Sonar-server') {
                    sh '${tool("sonar-scanner")}/bin/sonar-scanner -Dsonar.projectKey=Petclinic -Dsonar.sources=src'
                }
                dependencyCheck additionalArguments: '--scan ./ --format HTML --out .', odcInstallation: 'DP-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.html'
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_CREDENTIALS_ID) {
                        docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
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
