🧩 Integration of Trend Micro Artifact Scanner (TMAS) in Jenkins

This guide explains how to integrate Trend Micro Artifact Scanner (TMAS) into a Jenkins pipeline to automatically scan Docker images for vulnerabilities before pushing them to AWS ECR.

🪜 Step 1 – Install Docker and Jenkins on Your Server

1️⃣ Update your system

sudo apt update -y && sudo apt upgrade -y

2️⃣ Install Docker

sudo apt install docker.io -y

3️⃣ Enable and start Docker service

sudo systemctl enable docker
sudo systemctl start docker

4️⃣ Install Jenkins (on Ubuntu)

curl -fsSL https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update -y
sudo apt install openjdk-17-jdk -y
sudo apt install jenkins -y

5️⃣ Add Jenkins user to Docker group

sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

⚙️ Step 2 – Install TMAS CLI and AWS CLI

1️⃣ Install TMAS CLI

wget https://cli.artifactscan.cloudone.trendmicro.com/tmas-cli/latest/tmas-cli_Linux_x86_64.tar.gz
tar -xvf tmas-cli_Linux_x86_64.tar.gz
chmod +x tmas
sudo mv tmas /usr/local/bin/


✅ Test installation:

tmas version

2️⃣ Install AWS CLI v2

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install


✅ Verify:

aws --version

🔐 Step 3 – Configure AWS IAM User

Create an IAM user with the following permissions:

AmazonEC2ContainerRegistryFullAccess
AmazonEKSClusterPolicy (optional – for future EKS deployments)
AmazonECRPublicReadOnly

Generate AWS Access Key and Secret Access Key.

Configure locally:
aws configure

Provide:

AWS Access Key ID
AWS Secret Access Key
Default region → ap-south-1

🔑 Step 4 – Add Credentials in Jenkins

Navigate to:
Manage Jenkins → Credentials → Global → Add Credentials

a. AWS Credentials

Kind: Username and Password

Username: AWS_ACCESS_KEY
Password: AWS_SECRET_ACCESS_KEY
ID: aws-creds

b. TMAS API Key

Kind: Secret text
Secret: <YOUR_TMAS_API_KEY>
ID: tmas-api-key

🧾 Step 5 – Generate TMAS API Key

Login to Trend Micro Cloud One Console
.
Navigate to: Administration → API Keys.
Create a new API key with Master Admin permission.
Copy this key and add it to Jenkins credentials as tmas-api-key.

🐳 Step 6 – Create AWS ECR Repository

In AWS Console → ECR → Create Repository
Example name: myapp-repo

Clone your application code:

git clone https://github.com/mohitverma4109/two-tier-flask-app.git
cd two-tier-flask-app

🔌 Step 7 – Install Jenkins Plugins

Go to Manage Jenkins → Plugins → Available Plugins, and install:

✅ Pipeline
✅ Pipeline Stage View
✅ Git Plugin
✅ Docker Pipeline

Restart Jenkins after installation.

🚀 Step 8 – Jenkins Pipeline (Jenkinsfile)

Below is a complete Jenkins pipeline that:
Builds a Docker image
Scans it using TMAS
Pushes it to AWS ECR

pipeline {
    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '230541231671'
        ECR_REPO       = 'myapp-repo'
        IMAGE_TAG      = "latest"
        TMAS_URL       = 'https://container.trendmicro.com'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/mohitverma4109/two-tier-flask-app.git'
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region $AWS_REGION

                        aws ecr get-login-password --region $AWS_REGION | \
                          docker login --username AWS \
                          --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $ECR_REPO:$IMAGE_TAG .
                    docker tag $ECR_REPO:$IMAGE_TAG \
                      $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Scan Image with TMAS') {
            steps {
                withCredentials([string(credentialsId: 'tmas-api-key', variable: 'TMAS_API_KEY')]) {
                    sh '''
                        export TMAS_API_KEY=$TMAS_API_KEY
                        export TMAS_URL=$TMAS_URL

                        echo "Running TMAS scan on Docker image..."
                        tmas scan docker:$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG \
                          -V -M -S --region $AWS_REGION
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                '''
            }
        }
    }

    post {
        always {
            echo "✅ Pipeline execution completed. Check Trend Micro console for scan reports."
        }
    }
}

🎯 Outcome

✅ Pulls source code from GitHub
✅ Builds a Docker image
✅ Scans the image for vulnerabilities with TMAS
✅ Pushes the image to AWS ECR
✅ Results visible in Trend Micro Cloud One Console
