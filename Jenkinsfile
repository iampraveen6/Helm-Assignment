pipeline {
    agent any
    
    environment {
        REPO_URL = 'https://github.com/iampraveen6/Helm-Assignment.git'
        BRANCH = 'main'
        KUBE_CONFIG = credentials('kubeconfig') // Make sure this ID exists in Jenkins
        HELM_VERSION = '3.12.0'
        NAMESPACE = 'helm-assignment'
        RELEASE_NAME = 'myapp'
        DOCKER_CREDENTIALS = credentials('docker-creds') // Rename to match Jenkins credential
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: env.BRANCH]],
                    userRemoteConfigs: [[url: env.REPO_URL]]
                ])
            }
        }
        
        stage('Setup Tools') {
            steps {
                script {
                    sh """
                        if ! helm version --short | grep -q 'v${HELM_VERSION}'; then
                            curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
                            chmod 700 get_helm.sh
                            ./get_helm.sh --version v${HELM_VERSION}
                        fi
                    """
                    
                    withCredentials([file(credentialsId: env.KUBE_CONFIG, variable: 'KUBECONFIG_FILE')]) {
                        sh 'mkdir -p ~/.kube'
                        sh "cp ${KUBECONFIG_FILE} ~/.kube/config"
                    }
                    
                    sh 'helm version'
                    sh 'kubectl version'
                }
            }
        }
        
        // Rest of your stages (Lint, Deploy, Verify)...
    }
    
    post {
        always {
            node {
                script {
                    cleanWs()
                    if (currentBuild.result == 'FAILURE') {
                        emailext (
                            subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                            body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                                <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                            to: 'iampraveen6@gmail.com'
                        )
                    }
                }
            }
        }
        success {
            node {
                emailext (
                    subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    body: """<p>SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                        <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                    to: 'iampraveen6@gmail.com'
                )
            }
        }
    }
}