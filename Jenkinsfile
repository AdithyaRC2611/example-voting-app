
pipeline{
    agent any
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/AdithyaRC2611/example-voting-app.git', credentialsId: 'github'
            }
        }
        stage('Verify Checkout'){
            steps{
                sh 'ls -la'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=vote1 \
                    -Dsonar.projectKey=vote1 '''
                }
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){  
                       sh "ls"
                       sh "docker build -t vote ."
                       sh "docker tag vote adithyarc26/vote:latest "
                       sh "docker push adithyarc26/vote:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image adithyarc26/vote:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -itd -p 8081:80 adithyarc26/vote:latest'
                sh 'kubectl apply -f K8s/deployment.yml'
                sh 'kubectl apply -f K8s/service.yml'
                sh 'kubectl apply -f K8s/ingress.yml'
            }
        }
    }
}


