pipeline {
    agent any

    tools {
        jdk 'jdk11'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', changelog: false, poll: false, url: 'https://github.com/HarshwardhanBaghel/G6-Project.git'
            }
        }

        stage('Code compile') {
            steps {
                sh "mvn clean compile"
            }
        }

        stage('Unit Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('sonar-scanner') {
            steps {
                withSonarQubeEnv(credentialsId: 'new-ver-sonar', installationName: 'Sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Petclinic \
                        -Dsonar.exclusions=**/*.java \
                        -Dsonar.projectKey=Petclinic
                    '''
                }
            }
        }

        stage('OWASP Scan') {
            steps {
                // Run the Dependency-Check scan and generate an XML report
                dependencyCheck additionalArguments: '--scan ./ --format XML --out .', odcInstallation: 'DP-check'

                // Publish the generated XML report
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                sh "mvn clean install"
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-hub', toolName: 'docker') {
                        // Build Docker image
                        sh "docker build -t petclinicg ."
                    }
                }
            }
        }

        stage('Docker Tag & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-hub', toolName: 'docker') {
                        // Tag Docker image with a version (e.g., build number or commit ID)
                        def imageTag = "blackopsgun/pet-clinicg:latest"
                        sh "docker tag petclinicg $imageTag"

                        // Push to Docker Hub
                        sh "docker push $imageTag"
                    }
                }
            }
        }

        stage('Docker Deploy') {
            steps {
                script {
                    // Deploy the container
                    def containerName = "petg"
                    sh "docker run -d --name $containerName -p 8082:8080 blackopsgun/pet-clinicg:latest"

                    // Optionally, verify if the container is running (you could also use docker ps to confirm)
                    sh "docker ps -f name=$containerName"
                }
            }
        }
    }
}
