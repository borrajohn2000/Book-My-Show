pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/ASKFORTESAWS/Book-My-Show.git']]
                )

                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {

                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {

                sh '''
                cd bookmyshow-app

                ls -la

                if [ -f package.json ]; then

                    rm -rf node_modules package-lock.json

                    npm install

                else
                    echo "package.json not found!"
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {

                sh '''
                trivy fs . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {

                script {

                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {

                        sh '''
                        echo "Building Docker Image"

                        docker build --no-cache -t ASKFORTESAWS/bms:latest -f bookmyshow-app/Dockerfile bookmyshow-app

                        echo "Pushing Docker Image"

                        docker push ASKFORTESAWS/bms:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {

                sh '''
                echo "Stopping Old Container"

                docker stop bms || true

                docker rm bms || true

                echo "Running New Container"

                docker run -d --restart=always --name bms -p 3000:3000 ASKFORTESAWS/bms:latest

                sleep 5

                docker ps -a

                docker logs bms
                '''
            }
        }
    }

    post {

        always {

            emailext(
                attachLog: true,

                subject: "${currentBuild.result}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",

                body: """
                Project Name: ${env.JOB_NAME}

                Build Number: ${env.BUILD_NUMBER}

                Build URL: ${env.BUILD_URL}
                """,

                to: 'askfortesaws6@gmail.com',

                attachmentsPattern: 'trivyfs.txt'
            )
        }
    }
}
