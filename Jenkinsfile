pipeline {
    agent any

    environment {
        AWS_REGION        = 'ap-south-1'
        AWS_ACCOUNT_ID    = '773135747657'
        ECR_REPO          = 'seclock'
        ECR_REGISTRY      = '773135747657.dkr.ecr.ap-south-1.amazonaws.com'
        GITHUB_CREDS_ID   = 'github-pat'
        SONAR_SERVER_NAME = 'SonarQube'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt pytest httpx
                    pytest test_e2e.py
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                sh '''
                    SHORT_SHA=$(git rev-parse --short=7 HEAD)
                    docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${SHORT_SHA} .
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${SHORT_SHA}
                '''
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDS_ID}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        SHORT_SHA=$(git rev-parse --short=7 HEAD)
                        rm -rf gitops-temp
                        git clone https://${GIT_USER}:${GIT_PASS}@github.com/shamnadbakkerkt/seclock-gitops.git gitops-temp
                        cd gitops-temp/k8s
                        sed -i "s|image: ${ECR_REGISTRY}/${ECR_REPO}:.*|image: ${ECR_REGISTRY}/${ECR_REPO}:${SHORT_SHA}|g" deployment.yaml
                        git config user.name "jenkins-bot"
                        git config user.email "jenkins@seclock.internal"
                        git add deployment.yaml
                        git commit -m "chore(ci): update image to ${SHORT_SHA} [skip ci]" || echo "No changes to commit"
                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                rm -rf gitops-temp venv || true
            '''
        }
    }
}
