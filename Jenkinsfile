pipeline {
    agent any
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: 'jenkins-github-tomcat',
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
                    sh 'mvn test -DskipTests=true'
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                dir('backend') {
                    sh '''mvn sonar:sonar \
                        -Dsonar.projectKey=Fullstackproject \
                        -Dsonar.projectName=Fullstackproject \
                        -Dsonar.host.url=http://172.31.0.215:9000 \
                        -Dsonar.login=sqa_0846b472542dae98957418e79d8ff1d3772336cd \
                        -s /etc/maven/settings.xml'''
                }
            }
        }
        stage('Deploy to Nexus') {
            steps {
                dir('backend') {
                    sh 'mvn deploy -DskipTests=true -s /etc/maven/settings.xml'
                }
            }
        }
        stage('Docker Build & Push Frontend') {
            steps {
                dir('frontend') {
                    withDockerRegistry(credentialsId: 'Dockerhub', url: "") {
                        sh 'docker build -t acmanso/frontend:${BUILD_NUMBER} .'
                        sh 'docker push acmanso/frontend:${BUILD_NUMBER}'
                    }
                }
                sh 'docker image prune -f'
            }
        }
        stage('Docker Build & Push Backend') {
            steps {
                dir('backend') {
                    withDockerRegistry(credentialsId: 'Dockerhub', url: "") {
                        sh 'docker build -t acmanso/backend:${BUILD_NUMBER} .'
                        sh 'docker push acmanso/backend:${BUILD_NUMBER}'
                    }
                }
                sh 'docker image prune -f'
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'kubectl apply -f k8s/mysql-secret.yaml'
                    sh 'kubectl apply -f k8s/mysql-deployment.yaml'
                    sh "sed -i 's|acmanso/frontend:.*|acmanso/frontend:${BUILD_NUMBER}|' k8s/deployment-front.yaml"
                    sh "sed -i 's|acmanso/backend:.*|acmanso/backend:${BUILD_NUMBER}|' k8s/deployment-backend.yaml"
                    sh 'kubectl apply -f k8s/deployment-backend.yaml'
                    sh 'kubectl apply -f k8s/deployment-front.yaml'
                    sh 'kubectl rollout status deployment/frontend'
                    sh 'kubectl rollout status deployment/backend'
                }
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
