pipeline{
    agent {label 'linux' }

    tools{
        git 'Default'
    }

    environment {
        IMAGE_NAME = 'nitace/go-lang-app'
        IMAGE_TAG  = "v${env.BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = 'docker-hub-credentials'
        }
    stages{
        

        stage ('Build Image'){
            steps{
               sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage ('PUSH TO REGESTRY'){
           steps {
                withCredentials([usernamePassword(credentialsId: "${env.REGISTRY_CREDENTIALS}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            script {
                sh """
                    echo "$PASSWORD" | docker login -u "$USERNAME" --password-stdin
                     docker push ${IMAGE_NAME}:${IMAGE_TAG}
                """
                    }
                }
            }
        }
    }
}
