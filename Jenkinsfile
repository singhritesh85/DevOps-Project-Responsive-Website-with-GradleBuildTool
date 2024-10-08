pipeline{
    agent{
        node{
            label "Slave-1"
            customWorkspace "/home/jenkins/demo"
        }
    }
    environment{
        JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto.x86_64"
        PATH="$PATH:$JAVA_HOME/bin:/opt/gradle/bin"
    }
    parameters { 
        string(name: 'COMMIT_ID', defaultValue: '', description: 'Provide the Commit ID') 
        string(name: 'REPO_NAME', defaultValue: '', description: 'Provide the ECR Repository Name for Application Image')
        string(name: 'TAG_NAME', defaultValue: '', description: 'Provide the TAG Name')
        string(name: 'REPLICA_COUNT', defaultValue: '', description: 'Provide the number of Pods to be created')
    }
    stages{
        stage("Clone-Code"){
            steps{
                cleanWs()
                checkout scmGit(branches: [[name: "${COMMIT_ID}"]], extensions: [], userRemoteConfigs: [[url: 'https://github.com/singhritesh85/Java_Gradle_Responsive_Website.git']])
            }
        }
        stage("SonarQube-Analysis"){
            steps{
                script{
                    withSonarQubeEnv(installationName: 'SonarQube-Server', credentialsId: 'sonarqube') {
                        sh "gradle sonarqube"
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES')
                {
                    waitForQualityGate abortPipeline: true
                }
           }
        }
        stage("Gradle Build"){
            steps{
                sh 'gradle build'
            }
        }
        stage("OWASP Dependency Check"){
            steps{
                sh 'dependency-check.sh --scan . --out .'
            }
        }
        stage("Docker-Image"){
            steps{
                sh 'docker build -t ${REPO_NAME}:${TAG_NAME} . --no-cache'
                sh 'trivy image --exit-code 0 --severity MEDIUM,HIGH ${REPO_NAME}:${TAG_NAME}'
                //sh 'trivy image  --exit-code 1 --severity CRITICAL ${REPO_NAME}:${TAG_NAME}'
                sh 'aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 027330342406.dkr.ecr.us-east-2.amazonaws.com'
                sh 'docker push ${REPO_NAME}:${TAG_NAME}'
            }
        }
        stage("Deployment"){
            steps{
                //sh 'yes|argocd login argocd.singhritesh85.com --username admin --password Admin123'
                sh 'argocd login argocd.singhritesh85.com --username admin --password Admin123 --skip-test-tls  --grpc-web'
                sh 'argocd app create responsive-website --project default --repo https://github.com/singhritesh85/helm-repo-for-ArgoCD.git --path ./folo --dest-namespace responsive-website --dest-server https://kubernetes.default.svc --helm-set service.port=80 --helm-set image.repository=${REPO_NAME} --helm-set image.tag=${TAG_NAME} --helm-set replicaCount=${REPLICA_COUNT} --upsert'
                sh 'argocd app sync responsive-website'
            }
        }
    }
}
