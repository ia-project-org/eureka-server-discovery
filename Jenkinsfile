pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
        REPO_URL = 'https://github.com/ia-project-org/eureka-server-discovery.git'
        MANIFEST_REPO = 'https://github.com/ia-project-org/MicroservicesConfiguration.git'
        REGISTRY = 'minaouimh/ai'
        REGISTRY_CREDENTIAL = 'dockerhub'
        SONARQUBE_CREDENTIALS_ID = 'sonar'
    }
    stages {
        stage('clean work-space') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from SCM') {
            steps {
                git branch: 'main',
                credentialsId: 'github',
                url: REPO_URL
            }
        }
        stage('Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test Application') {
            steps{
                sh 'mvn test'
            }
        }
        stage('sonar analysis') {
            steps {
                withSonarQubeEnv(installationName: 'sonar' , credentialsId: SONARQUBE_CREDENTIALS_ID) {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('Quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false ,  credentialsId: SONARQUBE_CREDENTIALS_ID
                }
            }
        }
        stage('Docker Build and Push') {
            steps {
                script {
                    def repoName = REPO_URL.tokenize('/').last().replaceAll('.git', '')
                    docker.withRegistry('', REGISTRY_CREDENTIAL) {
                        DOCKER_IMAGE = docker.build("${REGISTRY}:${repoName}-${BUILD_NUMBER}")
                        DOCKER_IMAGE.push()
                    }
                }
            }
        }
         stage('Update Manifest') {
             steps {
                 script {
                     def id = 'eureka-server-discovery'
                     def imageTag = "${id}-${BUILD_NUMBER}"
                     withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'GITHUB_PASSWORD', usernameVariable: 'GITHUB_USERNAME')]) {
                         git credentialsId: 'github', url: MANIFEST_REPO, branch: 'main'
                         sh """
                             git status
                             sed -i 's|image: minaouimh/ai:.*|image: minaouimh/ai:${imageTag}|g' deployment/eureka-server/eureka-server-deployment.yaml
                             git config --global user.email "jenkins@example.com"
                             git config --global user.name "Jenkins"
                             git add deployment/eureka-server/eureka-server-deployment.yaml
                             git commit -m "Update image to ${imageTag}"
                             git push https://${GITHUB_USERNAME}:${GITHUB_PASSWORD}@github.com/ia-project-org/MicroservicesConfiguration.git main
                         """
                 }
         }
     }
 }
}
    post {
                always {
                    slackSend channel: '#jenkins-pipline-backend-notify',
                        message: """
                            *Pipeline Status:* ${currentBuild.currentResult}
                            *Job Name:* ${env.JOB_NAME}
                            *Build Number:* ${env.BUILD_NUMBER}
                            *Build URL:* ${BUILD_URL}
                            *Committer:* ${env.GIT_COMMITTER_NAME} (${env.GIT_COMMITTER_EMAIL})
                            *Commit Message:* ${env.GIT_COMMIT}
                            *Commit Time:* ${env.GIT_COMMITTER_DATE}
                            *Branch:* ${env.GIT_BRANCH}
                            """
                }
    }
}