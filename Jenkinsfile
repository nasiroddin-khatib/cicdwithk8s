pipeline {
    agent any

    environment {
        IMAGE_NAME = "nasir590/k8s-app"
        TAG = "v${BUILD_NUMBER}"
        EMAIL = "nasirkhatib01@gmail.com"
    }

    stages {

        stage('Clone Code') {
            steps {
                git 'https://github.com/nasiroddin-khatib/cicdwithk8s'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$TAG .'
            }
        }

        stage('Push Image') {
            steps {
                withDockerRegistry([ credentialsId: 'dockerhub-cred', url: 'https://index.docker.io/v1/' ]) {
                    sh 'docker push $IMAGE_NAME:$TAG'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                kubectl set image deployment k8s-deployment k8s-app=$IMAGE_NAME:$TAG
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    def status = sh(
                        script: "kubectl rollout status deployment k8s-deployment",
                        returnStatus: true
                    )

                    if (status != 0) {
                        error("Deployment Failed ❌")
                    }
                }
            }
        }

        stage('Apply Ingress') {
            when {
                expression { currentBuild.currentResult == 'SUCCESS' }
            }
            steps {
                sh 'kubectl apply -f ingress.yaml'
            }
        }

        stage('Mark as Stable') {
            when {
                expression { currentBuild.currentResult == 'SUCCESS' }
            }
            steps {
                withDockerRegistry([ credentialsId: 'dockerhub-cred', url: 'https://index.docker.io/v1/' ]) {
                    sh '''
                    docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:stable
                    docker push $IMAGE_NAME:stable
                    '''
                }
            }
        }
    }

    post {

        success {
            mail to: "${EMAIL}",
                 subject: "✅ Deployment SUCCESS",
                 body: """
Deployment Successful 🚀

Image: ${IMAGE_NAME}:${TAG}
Pods Status: Running ✅
Tagged as: stable
"""
        }

        failure {
            script {

                mail to: "${EMAIL}",
                     subject: "❌ Deployment FAILED",
                     body: """
Deployment Failed ❌

Image: ${IMAGE_NAME}:${TAG}
Starting automatic rollback...
"""

                echo "Deployment failed → starting rollback"

                // Rollback
                sh 'kubectl rollout undo deployment k8s-deployment'

                // Verify rollback
                def rbStatus = sh(
                    script: "kubectl rollout status deployment k8s-deployment",
                    returnStatus: true
                )

                if (rbStatus == 0) {
                    mail to: "${EMAIL}",
                         subject: "🔁 Rollback SUCCESS",
                         body: """
Rollback Completed Successfully ✅

System restored to previous stable version.
"""
                } else {
                    mail to: "${EMAIL}",
                         subject: "🚨 Rollback FAILED",
                         body: """
CRITICAL 🚨

Deployment failed ❌
Rollback also failed ❌

Immediate manual intervention required!
"""
                }
            }
        }
    }
}
