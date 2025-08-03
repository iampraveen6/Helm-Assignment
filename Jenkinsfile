pipeline {
    agent any
    
    environment {
        // General configuration
        REPO_URL = 'https://github.com/iampraveen6/Helm-Assignment.git'
        BRANCH = 'main'
        
        // Kubernetes/Helm configuration
        KUBE_CONFIG = credentials('kubeconfig') // Kubernetes config file credential ID in Jenkins
        HELM_VERSION = '3.12.0' // Helm version to use
        
        // Application specific
        NAMESPACE = 'helm-assignment'
        RELEASE_NAME = 'myapp'
        
        // Docker configuration (if needed)
        DOCKER_REGISTRY = 'your-registry.io'
        DOCKER_CREDENTIALS = credentials('docker-creds') // Docker registry credentials ID in Jenkins
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
                    // Install Helm if not present
                    sh """
                        if ! helm version --short | grep -q 'v${HELM_VERSION}'; then
                            curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
                            chmod 700 get_helm.sh
                            ./get_helm.sh --version v${HELM_VERSION}
                        fi
                    """
                    
                    // Configure kubectl with the provided kubeconfig
                    withCredentials([file(credentialsId: env.KUBE_CONFIG, variable: 'KUBECONFIG_FILE')]) {
                        sh 'mkdir -p ~/.kube'
                        sh "cp ${KUBECONFIG_FILE} ~/.kube/config"
                    }
                    
                    // Verify tools
                    sh 'helm version'
                    sh 'kubectl version'
                }
            }
        }
        
        stage('Lint Helm Charts') {
            steps {
                dir('helm-charts') {
                    script {
                        def charts = findFiles(glob: '*/Chart.yaml')
                        charts.each { chart ->
                            def chartDir = chart.path.replace('/Chart.yaml', '')
                            echo "Linting chart: ${chartDir}"
                            sh "helm lint ${chartDir}"
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Create namespace if it doesn't exist
                    sh "kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}"
                    
                    // Deploy each chart
                    dir('helm-charts') {
                        def charts = findFiles(glob: '*/Chart.yaml')
                        charts.each { chart ->
                            def chartDir = chart.path.replace('/Chart.yaml', '')
                            echo "Deploying chart: ${chartDir}"
                            
                            // Check if release exists
                            def releaseExists = sh(
                                script: "helm status ${RELEASE_NAME}-${chartDir} -n ${NAMESPACE} >/dev/null 2>&1 && echo 'exists' || echo 'not exists'",
                                returnStdout: true
                            ).trim()
                            
                            if (releaseExists == 'exists') {
                                // Upgrade existing release
                                sh """
                                    helm upgrade ${RELEASE_NAME}-${chartDir} ${chartDir} \
                                        --namespace ${NAMESPACE} \
                                        --install \
                                        --atomic \
                                        --timeout 5m
                                """
                            } else {
                                // Install new release
                                sh """
                                    helm install ${RELEASE_NAME}-${chartDir} ${chartDir} \
                                        --namespace ${NAMESPACE} \
                                        --atomic \
                                        --timeout 5m
                                """
                            }
                        }
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    // Check all deployments are ready
                    sh "kubectl get deployments -n ${NAMESPACE} -o wide"
                    
                    // Wait for deployments to be ready
                    sh """
                        kubectl wait --for=condition=available \
                            --timeout=300s \
                            -n ${NAMESPACE} \
                            --all deployments
                    """
                    
                    // Get services and ingress information
                    sh "kubectl get services -n ${NAMESPACE}"
                    sh "kubectl get ingress -n ${NAMESPACE} || true"
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Cleanup workspace
                cleanWs()
                
                // Send notification if build failed
                if (currentBuild.result == 'FAILURE') {
                    emailext (
                        subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                            <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                        to: 'iampraveen6@gmail.com',
                        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                    )
                }
            }
        }
        
        success {
            // Send success notification
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                    <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                to: 'iampraveen6@gmail.com'
            )
        }
    }
}