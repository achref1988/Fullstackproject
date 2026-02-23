pipeline {
    agent any

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'git@github.com:achref1988/Fullstackproject.git'
                sh 'ls -lah'
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh 'mvn clean package -DskipTests=true'
                }
            }
        }

        stage('Test Backend') {
            steps {
                dir('backend') {
                    sh 'mvn test'
                }
            }
        }

        stage('Docker Build & Push Frontend') {
            steps {
                dir('frontend') {
                    withDockerRegistry(credentialsId: 'Dockerhub', url: "") {
                        sh 'docker build -t achref1988/frontend:latest .'
                        sh 'docker push achref1988/frontend:latest'
                    }
                }
                sh 'docker image prune -f'
            }
        }

        stage('Docker Build & Push Backend') {
            steps {
                dir('backend') {
                    withDockerRegistry(credentialsId: 'Dockerhub', url: "") {
                        sh 'docker build -t achref1988/backend:latest .'
                        sh 'docker push achref1988/backend:latest'
                    }
                }
                sh 'docker image prune -f'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'kubectl apply -f k8s/mysql-secret.yaml'
                    sh 'kubectl apply -f k8s/mysql-pvc.yaml'
                    sh 'kubectl apply -f k8s/mysql-deployment.yaml'
                    sh 'kubectl apply -f k8s/deployment-backend.yaml'
                    sh 'kubectl apply -f k8s/deployment-front.yaml'
                    sh 'kubectl rollout status deployment/frontend'
                    sh 'kubectl rollout status deployment/backend'
                   /* sh INGRESS.YAML à faire ne pas oublier */                }
            }
        }

        stage('Verification') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'kubectl get pods'
                    sh 'kubectl get services'
                }
            }
        }
    }

    post {
        success {
            echo 'Déploiement réussi !'
        }
        failure {
            echo 'Echec ! Rollback...'
            withKubeConfig([credentialsId: 'kubeconfig']) {
                sh 'kubectl rollout undo deployment/frontend || true'
                sh 'kubectl rollout undo deployment/backend || true'
            }
        }
        always {
            sh 'docker image prune -f || true'
        }
    }

}

