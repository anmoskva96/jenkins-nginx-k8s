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
                sh 'curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3'
                sh 'chmod 700 get_helm.sh'
                sh './get_helm.sh'
                sh 'helm version'

                sleep time: 10
            }

        stage('Deploy Helm chart') {
            steps {
                script {
                    def helmReleaseName = 'nginx'

                    sh "helm install ${helmReleaseName} nginx-chart-v1/"

                    sleep time: 10

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
