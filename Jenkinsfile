pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        EMAIL_TO  = "achrefdevops2026@gmail.com"
    }

    stages {

        stage('Checkout Code') {
            steps {
                script {
                    try {
                        git branch: 'main',
                            credentialsId: 'jenkins-github-tomcat',
                            url: 'git@github.com:achref1988/Fullstackproject.git'
                        sh 'ls -lah'
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Checkout', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Checkout', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Build Backend') {
            steps {
                script {
                    try {
                        dir('backend') {
                            sh 'mvn clean package -DskipTests=true'
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Build', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Build', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Test Backend') {
            steps {
                script {
                    try {
                        dir('backend') {
                            sh 'mvn test -DskipTests=true'
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Test', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Test', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    try {
                        dir('backend') {
                            withSonarQubeEnv('sonarqube') {
                                sh '''mvn sonar:sonar \
                                    -Dsonar.projectKey=Fullstackproject \
                                    -Dsonar.projectName=Fullstackproject \
                                    -Dsonar.login=sqa_0846b472542dae98957418e79d8ff1d3772336cd \
                                    -s /etc/maven/settings.xml'''
                            }
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Sonar', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Sonar', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Quality', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Quality', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Deploy to Nexus') {
            steps {
                script {
                    try {
                        dir('backend') {
                            sh 'mvn deploy -DskipTests=true -s /etc/maven/settings.xml'
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Nexus', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Nexus', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Docker Build & Push Frontend') {
            steps {
                script {
                    try {
                        dir('frontend') {
                            withDockerRegistry(credentialsId: 'Dockerhub', url: "") {
                                sh 'docker build -t acmanso/frontend:${BUILD_NUMBER} .'
                                sh 'docker push acmanso/frontend:${BUILD_NUMBER}'
                            }
                        }
                        sh 'docker image prune -f'
                        currentBuild.description = updateStageStatus(currentBuild.description, 'DockerFront', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'DockerFront', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Docker Build & Push Backend') {
            steps {
                script {
                    try {
                        dir('backend') {
                            withDockerRegistry(credentialsId: 'Dockerhub', url: "") {
                                sh 'docker build -t acmanso/backend:${BUILD_NUMBER} .'
                                sh 'docker push acmanso/backend:${BUILD_NUMBER}'
                            }
                        }
                        sh 'docker image prune -f'
                        currentBuild.description = updateStageStatus(currentBuild.description, 'DockerBack', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'DockerBack', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        withKubeConfig([credentialsId: 'kubeconfig']) {
                            sh 'kubectl apply -f k8s/mysql-secret.yaml'
                            sh 'kubectl apply -f k8s/mysql-deployment.yaml'
                            sh "sed -i 's|acmanso/frontend:.*|acmanso/frontend:${BUILD_NUMBER}|' k8s/deployment-front.yaml"
                            sh "sed -i 's|acmanso/backend:.*|acmanso/backend:${BUILD_NUMBER}|' k8s/deployment-backend.yaml"
                            sh 'kubectl apply -f k8s/deployment-backend.yaml'
                            sh 'kubectl apply -f k8s/deployment-front.yaml'
                            sh 'kubectl apply -f k8s/ingress.yaml'
                            sh 'kubectl rollout status deployment/frontend'
                            sh 'kubectl rollout status deployment/backend'
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Deploy', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Deploy', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Verification') {
            steps {
                script {
                    try {
                        withKubeConfig([credentialsId: 'kubeconfig']) {
                            sh 'kubectl get pods'
                            sh 'kubectl get services'
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Verification', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Verification', 'FAILED')
                        throw e
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def statusMap = parseStageStatus(currentBuild.description ?: '')

                def stagesTable = """
                <table style="border-collapse: collapse; width: 100%; font-family: Arial;">
                    <thead>
                        <tr style="background-color: #4CAF50; color: white;">
                            <th style="padding: 10px;">Stage</th>
                            <th style="padding: 10px;">Status</th>
                            <th style="padding: 10px;">Icon</th>
                        </tr>
                    </thead>
                    <tbody>
                        ${createStageRow('Checkout Code', statusMap.get('Checkout', 'PENDING'))}
                        ${createStageRow('Build Backend', statusMap.get('Build', 'PENDING'))}
                        ${createStageRow('Test Backend', statusMap.get('Test', 'PENDING'))}
                        ${createStageRow('SonarQube Analysis', statusMap.get('Sonar', 'PENDING'))}
                        ${createStageRow('Quality Gate', statusMap.get('Quality', 'PENDING'))}
                        ${createStageRow('Deploy to Nexus', statusMap.get('Nexus', 'PENDING'))}
                        ${createStageRow('Docker Frontend', statusMap.get('DockerFront', 'PENDING'))}
                        ${createStageRow('Docker Backend', statusMap.get('DockerBack', 'PENDING'))}
                        ${createStageRow('Deploy to Kubernetes', statusMap.get('Deploy', 'PENDING'))}
                        ${createStageRow('Verification', statusMap.get('Verification', 'PENDING'))}
                    </tbody>
                </table>
                """

                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def statusIcon = buildStatus == 'SUCCESS' ? '✅' : '❌'

                emailext(
                    subject: "${statusIcon} ${buildStatus}: ${JOB_NAME} #${BUILD_NUMBER}",
                    body: stagesTable,
                    to: "${EMAIL_TO}",
                    mimeType: 'text/html'
                )
            }

            echo 'Cleaning workspace...'
            cleanWs()
        }

        failure {
            withKubeConfig([credentialsId: 'kubeconfig']) {
                sh 'kubectl rollout undo deployment/frontend || true'
                sh 'kubectl rollout undo deployment/backend || true'
            }
        }
    }
}

// ==================== FONCTIONS PERSONNALISÉES ====================

def updateStageStatus(String description, String stageName, String status) {
    description = description ?: ''
    def marker = "${stageName}:${status}"
    description = description.replaceAll("${stageName}:(SUCCESS|FAILED|PENDING)", '')
    description = description.replaceAll(',+', ',').replaceAll('^,|,$', '')
    if (description) {
        return "${description},${marker}"
    } else {
        return marker
    }
}

def parseStageStatus(String description) {
    def statusMap = [:]
    if (!description) return statusMap
    description.split(',').each { item ->
        def parts = item.trim().split(':')
        if (parts.size() == 2) statusMap[parts[0]] = parts[1]
    }
    return statusMap
}

def createStageRow(String stageName, String status) {
    def color = ''
    def icon = ''
    switch(status) {
        case 'SUCCESS': color='#4CAF50'; icon='✅'; break
        case 'FAILED':  color='#f44336'; icon='❌'; break
        case 'PENDING': color='#9E9E9E'; icon='⏳'; break
        default: color='#9E9E9E'; icon='❓'
    }
    return """
        <tr style="background-color: #f9f9f9;">
            <td style="padding: 10px; border: 1px solid #ddd;">${stageName}</td>
            <td style="padding: 10px; border: 1px solid #ddd; color: ${color}; font-weight: bold;">${status}</td>
            <td style="padding: 10px; border: 1px solid #ddd; text-align: center; font-size: 20px;">${icon}</td>
        </tr>
    """
}
