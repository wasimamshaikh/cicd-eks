pipeline {

    agent any
/*
	tools {
        maven "maven3"
    }
*/
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        ECR_REGISTRY = '334381385047.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg'
        ECR_REPO_NAME = 'vprofileappimg'
        K8S_NAMESPACE = 'prod'
        HELM_RELEASE_NAME = 'vprofile-stack'
        HELM_CHART_PATH = '/helm/vprofilecharts'
    }

    stages{

        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps{
              script {
                sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest ."
              }
            }
        }

        stage('Docker Push to ECR') {
          steps{
            script {
              sh "docker push ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest"
            }
          }
        }

        stage('Kubernetes Deploy') {
	        steps {
	            script {
                    sh "aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name my-first-cluster"
                    sh "helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_CHART_PATH} --namespace ${K8S_NAMESPACE} --set image.repository=${ECR_REGISTRY}/${ECR_REPO_NAME},image.tag=latest"
                    }
            }
        }

    }

}