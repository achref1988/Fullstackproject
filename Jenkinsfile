pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        EMAIL_TO  = "achrefdevops2026@gmail.com"
    }

    stages {

        // ─── CHECKOUT ────────────────────────────────────────────────
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

        // ─── CI BACKEND ──────────────────────────────────────────────
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

        // ─── CD : DOCKER BUILD & PUSH (via Ansible) ──────────────────
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

        // ─── CD : UPLOAD FRONTEND TO NEXUS ───────────────────────────
        stage('Upload Frontend to Nexus') {
            steps {
                script {
                    try {
                        dir('frontend') {
                            sh 'npm install'
                            sh 'npm run build'
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

        // ─── DEPLOY : KUBERNETES (via Ansible) ───────────────────────
        stage('Continuous Deployment') {
            steps {
                script {
                    try {
                        sh """
                            ansible-playbook -i inventory.yml playbook_continousdeployment.yml \
                                -e "IMAGE_TAG=${IMAGE_TAG}"
                        """
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Deploy', 'SUCCESS')
                    } catch (Exception e) {
                        currentBuild.description = updateStageStatus(currentBuild.description, 'Deploy', 'FAILED')
                        throw e
                    }
                }
            }
        }

        // ─── VERIFICATION ─────────────────────────────────────────────
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

    // ─── POST ACTIONS ─────────────────────────────────────────────────
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
                        ${createStageRow('Checkout Code',            statusMap.get('Checkout',     'PENDING'))}
                        ${createStageRow('Build Backend',            statusMap.get('Build',        'PENDING'))}
                        ${createStageRow('Test Backend',             statusMap.get('Test',         'PENDING'))}
                        ${createStageRow('SonarQube Analysis',       statusMap.get('Sonar',        'PENDING'))}
                        ${createStageRow('Quality Gate',             statusMap.get('Quality',      'PENDING'))}
                        ${createStageRow('Deploy Backend to Nexus',  statusMap.get('NexusBack',    'PENDING'))}
                        ${createStageRow('Docker Frontend',          statusMap.get('DockerFront',  'PENDING'))}
                        ${createStageRow('Upload Frontend to Nexus', statusMap.get('NexusFront',   'PENDING'))}
                        ${createStageRow('Docker Backend',           statusMap.get('DockerBack',   'PENDING'))}
                        ${createStageRow('Deploy to Kubernetes',     statusMap.get('Deploy',       'PENDING'))}
                        ${createStageRow('Verification',             statusMap.get('Verification', 'PENDING'))}
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
