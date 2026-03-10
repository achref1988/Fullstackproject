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

        stage('Build') {
            parallel {
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

                stage('Build Frontend') {
                    steps {
                        script {
                            try {
                                dir('frontend') {
                                    sh 'npm install'
                                    sh 'npm run build'
                                }
                                currentBuild.description = updateStageStatus(currentBuild.description, 'BuildFront', 'SUCCESS')
                            } catch (Exception e) {
                                currentBuild.description = updateStageStatus(currentBuild.description, 'BuildFront', 'FAILED')
                                throw e
                            }
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            parallel {
                stage('SonarQube Backend') {
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
                                        timeout(time: 5, unit: 'MINUTES') {
                                            waitForQualityGate abortPipeline: true
                                        }
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

                stage('SonarQube Frontend') {
                    steps {
                        script {
                            try {
                                dir('frontend') {
                                    withSonarQubeEnv('sonarqube') {
                                        def scannerHome = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                                        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                                            sh """
                                                ${scannerHome}/bin/sonar-scanner \
                                                -Dsonar.projectKey=Fullstackproject-frontend \
                                                -Dsonar.projectName=Fullstackproject-frontend \
                                                -Dsonar.sources=src \
											                        	-Dsonar.host.url=http://172.31.0.215:9000 \
                                                -Dsonar.login=\$SONAR_TOKEN
                                            """
                                        } 
                                     
                                        timeout(time: 5, unit: 'MINUTES') {
                                            waitForQualityGate abortPipeline: true
                                        }
                                    } // ← ferme withSonarQubeEnv
                                } // ← ferme dir
                                currentBuild.description = updateStageStatus(currentBuild.description, 'SonarFront', 'SUCCESS')
                            } catch (Exception e) {
                                currentBuild.description = updateStageStatus(currentBuild.description, 'SonarFront', 'FAILED')
                                throw e
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy Backend to Nexus') {
            steps {
                script {
                    try {
                        dir('backend') {
                            sh 'mvn deploy -DskipTests=true -s /etc/maven/settings.xml'
                        }
                        currentBuild.description = updateStageStatus(currentBuild.description, 'NexusBack', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'NexusBack', 'FAILED')
                        throw e
                    }
                }
            }
        }

        stage('Delivery & Nexus Frontend') {
            parallel {
                stage('Continuous Delivery') {
                    steps {
                        script {
                            try {
                                withCredentials([usernamePassword(
                                    credentialsId: 'Dockerhub',
                                    usernameVariable: 'DOCKER_USER',
                                    passwordVariable: 'DOCKER_PASS'
                                )]) {
                                    sh """
                                        ansible-playbook -i inventory.yml playbook_continousdelivery.yml \
                                            -e "DOCKER_USER=\$DOCKER_USER" \
                                            -e "DOCKER_PASS=\$DOCKER_PASS" \
                                            -e "IMAGE_TAG=${IMAGE_TAG}"
                                    """
                                }
                                currentBuild.description = updateStageStatus(currentBuild.description, 'DockerFront', 'SUCCESS')
                                currentBuild.description = updateStageStatus(currentBuild.description, 'DockerBack', 'SUCCESS')
                            } catch (Exception e) {
                                currentBuild.description = updateStageStatus(currentBuild.description, 'DockerFront', 'FAILED')
                                currentBuild.description = updateStageStatus(currentBuild.description, 'DockerBack', 'FAILED')
                                throw e
                            }
                        }
                    }
                }

                stage('Upload Frontend to Nexus') {
                    steps {
                        script {
                            try {
                                dir('frontend') {
                                    sh "zip -r frontend-${IMAGE_TAG}.zip dist/"
                                }
                                nexusArtifactUploader(
                                    nexusVersion: 'nexus3',
                                    protocol: 'http',
                                    nexusUrl: '172.31.0.215:8081',
                                    repository: 'frontend-raw',
                                    credentialsId: 'Nexus',
                                    groupId: 'com.devops',
                                    version: "0.0.1-${IMAGE_TAG}",
                                    artifacts: [
                                        [
                                            artifactId: 'frontend',
                                            classifier: '',
                                            file: "frontend/frontend-${IMAGE_TAG}.zip",
                                            type: 'zip'
                                        ]
                                    ]
                                )
                                currentBuild.description = updateStageStatus(currentBuild.description, 'NexusFront', 'SUCCESS')
                            } catch (Exception e) {
                                currentBuild.description = updateStageStatus(currentBuild.description, 'NexusFront', 'FAILED')
                                throw e
                            }
                        }
                    }
                }
            }
        }

        stage('Continuous Deployment') {
            steps {
                script {
                    try {
                        // STEP 1 : Pull images sur DockerSRV via Ansible
                        sh """
                            ansible-playbook -i inventory.yml playbook_continousdeployment.yml \
                                -e "IMAGE_TAG=${IMAGE_TAG}"
                        """

                        // STEP 2 : Delete + Redeploy sur Kubernetes
                        withKubeConfig([credentialsId: 'kubeconfig']) {

                           
                            // DEPLOY FRESH
                            sh 'kubectl apply -f k8s/mysql-secret.yaml'
                            sh 'kubectl apply -f k8s/mysql-deployment.yaml'
                            sh "sed -i 's|acmanso/frontend:.*|acmanso/frontend:${IMAGE_TAG}|' k8s/deployment-front.yaml"
                            sh "sed -i 's|acmanso/backend:.*|acmanso/backend:${IMAGE_TAG}|'   k8s/deployment-backend.yaml"
                            sh 'kubectl apply -f k8s/deployment-backend.yaml'
                            sh 'kubectl apply -f k8s/deployment-front.yaml'
                            sh 'kubectl apply -f k8s/ingress.yaml'

                            // WAIT FOR ROLLOUT
                            sh 'kubectl rollout status deployment/backend  --timeout=120s'
                            sh 'kubectl rollout status deployment/frontend --timeout=120s'
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
                            sh 'kubectl get ingress'
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
                        ${createStageRow('Checkout Code',            statusMap.get('Checkout',      'PENDING'))}
                        ${createStageRow('Build Backend',            statusMap.get('Build',         'PENDING'))}
                        ${createStageRow('Build Frontend',           statusMap.get('BuildFront',    'PENDING'))}
                        ${createStageRow('SonarQube Backend',        statusMap.get('Sonar',         'PENDING'))}
                        ${createStageRow('SonarQube Frontend',       statusMap.get('SonarFront',    'PENDING'))}
                        ${createStageRow('Quality Gate Backend',     statusMap.get('Quality',       'PENDING'))}
                        ${createStageRow('Quality Gate Frontend',    statusMap.get('QualityFront',  'PENDING'))}
                        ${createStageRow('Deploy Backend to Nexus',  statusMap.get('NexusBack',     'PENDING'))}
                        ${createStageRow('Docker Frontend',          statusMap.get('DockerFront',   'PENDING'))}
                        ${createStageRow('Docker Backend',           statusMap.get('DockerBack',    'PENDING'))}
                        ${createStageRow('Upload Frontend to Nexus', statusMap.get('NexusFront',    'PENDING'))}
                        ${createStageRow('Deploy to Kubernetes',     statusMap.get('Deploy',        'PENDING'))}
                        ${createStageRow('Verification',             statusMap.get('Verification',  'PENDING'))}
                    </tbody>
                </table>
                """

                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def statusIcon  = buildStatus == 'SUCCESS' ? '✅' : '❌'

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
                sh 'kubectl rollout undo deployment/backend  || true'
            }
        }
    }
}

// ==================== FONCTIONS PERSONNALISÉES ====================

def updateStageStatus(String description, String stageName, String status) {
    description = description ?: ''
    def marker = "${stageName}:${status}"
    description = description.replaceAll("${stageName}:(SUCCESS|FAILED|PENDING)", '')
    description = description.replaceAll(',+', ',').replaceAll('^,|,\$', '')
    return description ? "${description},${marker}" : marker
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
    def icon  = ''
    switch(status) {
        case 'SUCCESS': color = '#4CAF50'; icon = '✅'; break
        case 'FAILED':  color = '#f44336'; icon = '❌'; break
        case 'PENDING': color = '#9E9E9E'; icon = '⏳'; break
        default:        color = '#9E9E9E'; icon = '❓'
    }
    return """
        <tr style="background-color: #f9f9f9;">
            <td style="padding: 10px; border: 1px solid #ddd;">${stageName}</td>
            <td style="padding: 10px; border: 1px solid #ddd; color: ${color}; font-weight: bold;">${status}</td>
            <td style="padding: 10px; border: 1px solid #ddd; text-align: center; font-size: 20px;">${icon}</td>
        </tr>
    """
}
