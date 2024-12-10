/*
Jenkins CI/CD Pipeline for a Microservice Deployment

This pipeline automates the process of building, testing, analyzing, containerizing, and deploying a microservice application. 
It integrates with AWS services like CodeCommit, ECR, Kubernetes, and Terraform to ensure smooth and efficient infrastructure 
and application lifecycle management.

Stages Overview:
1. **Checkout**:
   - Retrieves the application source code securely from an AWS CodeCommit repository using environment variables to 
     ensure sensitive details are abstracted.

2. **Terraform Apply**:
   - Provisions or updates infrastructure resources using Terraform. 
   - Ensures that the required Kubernetes clusters, namespaces, and other infrastructure are ready before deployment.

3. **Maven Build**:
   - Compiles and builds the Java application using Maven, skipping tests to save time during the build process.

4. **Run Unit Tests**:
   - Executes unit tests to validate the functionality of the application.
   - Publishes test results for analysis.

5. **SonarQube Analysis**:
   - Conducts static code analysis to identify code quality issues, security vulnerabilities, and technical debt using SonarQube.

6. **Build Docker Image**:
   - Creates a Docker image of the application and tags it with the build number for versioning.

7. **Push to ECR**:
   - Pushes the Docker image to an Amazon Elastic Container Registry (ECR) for secure storage and deployment.

8. **Deploy to Development**:
   - Deploys the Docker image to a Kubernetes development environment using Helm charts for configuration.

9. **Deploy to Staging**:
   - Deploys the Docker image to a Kubernetes staging environment for further validation.

10. **Determine Active Deployment**:
    - Checks the current active deployment in the production environment (blue or green) for a blue-green deployment strategy.

11. **Blue-Green Deployment to Production**:
    - Performs a blue-green deployment to minimize downtime and ensure a smooth transition for production updates.
    - Updates the Kubernetes service to point to the new deployment and verifies its health before cleaning up the old deployment.

Post-Stage:
- Cleans up Docker images from the Jenkins environment after the pipeline execution to conserve space.

Key Features:
- **Infrastructure as Code (IaC)**: Uses Terraform to manage and provision infrastructure.
- **Blue-Green Deployment**: Ensures zero-downtime production updates by deploying new versions alongside the existing one.
- **Kubernetes Integration**: Deploys microservices to Kubernetes clusters for development, staging, and production environments.
- **Secure Code Management**: Retrieves source code securely from AWS CodeCommit.
- **Quality Assurance**: Includes unit tests and static code analysis to maintain code quality and reliability.
*/

pipeline {
    agent any
    environment {
        AWS_CREDENTIALS = credentials('aws-credentials-id')
        CODECOMMIT_REPO = 'my-codecommit-repo'
        CODECOMMIT_BRANCH = 'main'
        CODECOMMIT_REGION = 'us-east-1'
        ECR_REPO = '123456789.dkr.ecr.region.amazonaws.com/my-microservice'
        DOCKER_IMAGE = ''
        DEV_CLUSTER_CONTEXT = 'dev-cluster-context'
        STAGING_CLUSTER_CONTEXT = 'staging-cluster-context'
        PROD_CLUSTER_CONTEXT = 'prod-cluster-context'
        CURRENT_DEPLOYMENT = ''
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    sh """
                        aws codecommit get-file --repository-name ${CODECOMMIT_REPO} \
                        --branch-name ${CODECOMMIT_BRANCH} \
                        --region ${CODECOMMIT_REGION} \
                        --output json > repo.json
                        jq -r .fileContents repo.json | base64 -d > repo.zip
                        unzip repo.zip
                    """
                }
            }
        }
        stage('Terraform Apply') {
            steps {
                script {
                    sh """
                        cd terraform
                        terraform init
                        terraform plan -out=tfplan
                        terraform apply -auto-approve tfplan
                    """
                }
            }
        }
        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Run Unit Tests') {
            steps {
                sh 'mvn test'
                junit '**/target/surefire-reports/*.xml'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQube Scanner'
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=my-microservice -Dsonar.sources=src"
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    DOCKER_IMAGE = "${ECR_REPO}:${env.BUILD_NUMBER}"
                }
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }
        stage('Push to ECR') {
            steps {
                script {
                    sh "$(aws ecr get-login --no-include-email --region ${CODECOMMIT_REGION})"
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }
        stage('Deploy to Development') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    kubernetesDeploy('dev', DEV_CLUSTER_CONTEXT, 'values-dev.yaml')
                }
            }
        }
        stage('Deploy to Staging') {
            when {
                branch 'staging'
            }
            steps {
                script {
                    kubernetesDeploy('staging', STAGING_CLUSTER_CONTEXT, 'values-staging.yaml')
                }
            }
        }
        stage('Determine Active Deployment') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def currentColor = sh(script: "kubectl get svc my-service --context ${PROD_CLUSTER_CONTEXT} -o=jsonpath='{.spec.selector.app}'", returnStdout: true).trim()
                    CURRENT_DEPLOYMENT = (currentColor == 'blue') ? 'green' : 'blue'
                }
            }
        }
        stage('Blue-Green Deployment to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def newDeployment = (CURRENT_DEPLOYMENT == 'blue') ? 'green' : 'blue'
                    sh """
                        helm upgrade --install my-service-${newDeployment} ./helm-chart \
                        --namespace production --context ${PROD_CLUSTER_CONTEXT} -f values-prod.yaml \
                        --set image.tag=${env.BUILD_NUMBER} --set app=${newDeployment}
                    """
                    sh "kubectl rollout status deployment/my-service-${newDeployment} --context ${PROD_CLUSTER_CONTEXT} --timeout=120s"
                    sh "curl -f http://my-service-${newDeployment}.prod.company.com/health || exit 1"
                    sh "kubectl patch svc my-service --context ${PROD_CLUSTER_CONTEXT} -p '{\"spec\":{\"selector\":{\"app\":\"${newDeployment}\"}}}'"
                    sh "kubectl delete deployment my-service-${CURRENT_DEPLOYMENT} --context ${PROD_CLUSTER_CONTEXT}"
                }
            }
        }
    }
    post {
        always {
            sh 'docker rmi ${DOCKER_IMAGE}'
        }
    }
}

def kubernetesDeploy(envName, clusterContext, valuesFile) {
    sh "kubectl config use-context ${clusterContext}"
    sh "helm upgrade --install my-service ./helm-chart --namespace ${envName} -f ${valuesFile}"
}
