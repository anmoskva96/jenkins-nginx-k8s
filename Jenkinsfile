pipeline {
    agent any

    stages {
        stage('Clone repository') {
            steps {
                git 'https://github.com/anmoskva96/jenkins-nginx-k8s.git'
            }
        }

        stage('Install Helm') {
            steps {
                sh 'curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null'
                sh 'apt-get install apt-transport-https --yes'
                sh 'echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list'
                sh 'apt-get update'
                sh 'apt-get install helm'
            }
        }


        stage('Deploy Helm chart') {
            steps {
                script {
                    def helmReleaseName = 'nginx'

                    sh "helm install ${helmReleaseName} nginx-chart-v1/"

                    sleep time: 60

                    sh "kubectl port-forward svc/nginx-service 32080:80 &"

                    sleep time: 10

                    def responseCode = sh(script: "curl -sL -w '%{http_code}' http://localhost:32080 -o /dev/null", returnStatus: true)

                    if (responseCode == 200) {
                        echo "Nginx доступен"
                    } else {
                        error "Невозможно получить доступ к Nginx серверу"
                    }
                }
            }
        }
    }
}
