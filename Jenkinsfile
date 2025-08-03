pipeline {
    agent any

    /* ---------- Environment variables ---------- */
    environment {
        /* Docker Hub images */
        FRONTEND_IMAGE = 'iampraveen6/learner-report-frontend'
        BACKEND_IMAGE  = 'iampraveen6/learner-report-backend'

        /* Relative paths inside this repo */
        CHART_DIR  = './learner-report-chart'
        FRONT_DIR  = './learnerReportCS_frontend'
        BACK_DIR   = './learnerReportCS_backend'

        /* Kubeconfig supplied via Jenkins credential */
        KUBECONFIG = credentials('KUBECONFIG_FILE')
    }

    /* ---------- Global options ---------- */
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {

        /* ---------- 1. Checkout source repos ---------- */
        stage('Clone Sources') {
            steps {
                /* Main repo already checked out by Jenkins */
                sh '''
                    echo "Cloning frontend & backend repositories ..."
                    [ -d learnerReportCS_frontend ] || \
                        git clone https://github.com/UnpredictablePrashant/learnerReportCS_frontend.git
                    [ -d learnerReportCS_backend ] || \
                        git clone https://github.com/UnpredictablePrashant/learnerReportCS_backend.git
                '''
            }
        }

        /* ---------- 2. Docker Build & Push ---------- */
        stage('Docker Build & Push') {
            parallel {
                stage('Frontend') {
                    steps {
                        dir(env.FRONT_DIR) {
                            script {
                                def img = docker.build("${FRONTEND_IMAGE}:${BUILD_NUMBER}")
                                docker.withRegistry('', 'DOCKER_HUB_CREDS') {
                                    img.push()
                                    img.push('latest')
                                }
                            }
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        dir(env.BACK_DIR) {
                            script {
                                def img = docker.build("${BACKEND_IMAGE}:${BUILD_NUMBER}")
                                docker.withRegistry('', 'DOCKER_HUB_CREDS') {
                                    img.push()
                                    img.push('latest')
                                }
                            }
                        }
                    }
                }
            }
        }

        /* ---------- 3. MongoDB Secret ---------- */
        stage('Create MongoDB Secret') {
            steps {
                withCredentials([
                    string(credentialsId: 'MONGODB_USER',     variable: 'DB_USER'),
                    string(credentialsId: 'MONGODB_PASSWORD', variable: 'DB_PASS'),
                    string(credentialsId: 'MONGODB_URI',      variable: 'DB_URI')
                ]) {
                    sh '''
                        kubectl create secret generic mongodb-secret \
                          --from-literal=username=${DB_USER} \
                          --from-literal=password=${DB_PASS} \
                          --from-literal=uri=${DB_URI} \
                          --dry-run=client -o yaml | \
                        kubectl apply --kubeconfig=${KUBECONFIG} -f -
                    '''
                }
            }
        }

        /* ---------- 4. Helm Deploy ---------- */
        stage('Helm Deploy') {
            steps {
                dir(env.CHART_DIR) {
                    sh """
                        helm upgrade --install learner-report . \
                          --set backend.tag=${BUILD_NUMBER} \
                          --set frontend.tag=${BUILD_NUMBER} \
                          --kubeconfig=${KUBECONFIG}
                    """
                }
            }
        }

        /* ---------- 5. Quick Health-Check ---------- */
        stage('Verify') {
            steps {
                sh '''
                    kubectl get pods --selector=app.kubernetes.io/name=learner-report \
                        --kubeconfig=${KUBECONFIG}
                    kubectl get svc \
                        --kubeconfig=${KUBECONFIG}
                '''
            }
        }
    }

    /* ---------- Post-build actions ---------- */
    post {
        success {
            echo '✅ MERN stack deployed successfully!'
        }
        failure {
            echo '❌ Pipeline failed – see logs for details.'
            emailext (
                subject: "Build ${BUILD_NUMBER} FAILED",
                body: "Check ${BUILD_URL}",
                to: 'dev-team@example.com'
            )
        }
    }
}