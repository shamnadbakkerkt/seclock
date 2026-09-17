pipeline {
    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = credentials('aws-account-id') // Or your AWS Account ID
        ECR_REPO_NAME  = 'seclock'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG      = "${GIT_COMMIT.take(7)}"
        GITOPS_REPO    = 'github.com/shamnadbakkerkt/seclock-gitops.git'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Python Test Suite') {
            steps {
                sh '''
                    python3 -m venv test-env
                    . test-env/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt pytest
                    python test_e2e.py
                    deactivate
                    rm -rf test-env
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh '/opt/sonar-scanner/bin/sonar-scanner'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} .
                    docker tag ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest
                """
            }
        }

        stage('Push to Amazon ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest
                """
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                        git config --global user.email "ci-bot@seclock.local"
                        git config --global user.name "Jenkins CI"
                        
                        rm -rf seclock-gitops
                        git clone https://${GIT_TOKEN}@${GITOPS_REPO}
                        cd seclock-gitops/overlays/prod
                        
                        # Update the deployment image tag
                        sed -i 's|image: .*|image: ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}|g' deployment.yaml
                        
                        git add deployment.yaml
                        git commit -m "Automated build: Update seclock image to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} || true"
            cleanWs()
        }
    }
}
