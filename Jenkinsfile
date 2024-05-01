pipeline {
    agent any

    stages {
        stage('Clone repository') {
            steps {
                git 'https://github.com/anmoskva96/jenkins-nginx-k8s.git'
            }
        }

        stage('Deploy Helm chart') {
            steps {
                script {
                    def helmReleaseName = 'nginx'

                    sh "helm install ${helmReleaseName} nginx-chart-v1/

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
