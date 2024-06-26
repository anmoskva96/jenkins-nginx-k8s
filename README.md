# Задание на DevOps


***
## Part 1. Развернуть minikube

### 1.1 Установка и запуск minikube на Ubuntu 20.04.6 LTS

> Примечание: Перед установкой minikube на локальной машине должны быть установлены kubectl и Hypervisor. В моем случае они установлены, Hypervisor использую VirtualBox

Для установки minikube воспользуемся документацией на сайте https://kubernetes.io/ru/docs/tasks/tools/install-minikube/

Я установил minikube с помощью прямой ссылки.

Запустим minikube с помощью команды `minikube start --vm-driver=virtualbox --cpus 2 --memory 4096 --disk-size 15g`

## Part 2. Написать Helm чарт, который устанавливает последнюю версию Nginx, должен быть доступен по порту 32080 на ноде Kubernetes

### 1.1 Установка helm

Установим helm согласно документации на сайте https://helm.sh/docs/intro/install/

Я воспользовался следующими командами:

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update

sudo apt-get install helm
```
Можно проверить, что helm установился, командой `helm version` 

### 1.2 Написание helm chart

Выполним команду `helm create nginx-chart-v1`, которая создас структуру каталогов и файлов для helm чарта nginx-server

Можно удалить лишние файлы, такие как: _helpers.tpl, hpa.yaml, ingress.yaml,NOTES.txt, serviceaccount.yaml и папку tests

Содержимое файла __values.yaml:__

```yaml
replicaCount: 2
image       :
  repository: nginx
  tag       : latest

service     :
  type      : NodePort
```

Содержимое файла __Chart.yaml:__

```yaml
apiVersion : v2
name       : nginx-chart-v1
description: A Helm chart for Kubernetes
type       : application
version    : 0.1.0
appVersion : "1.16.0"
```

Содержимое файла __deployment.yaml:__

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: https
              containerPort: 80
```

Содержимое файла __service.yaml:__

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
  labels:
    app: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - protocol  : TCP
      port      : 32080
      targetPort: 80
  type: {{ .Values.service.type }}
```

Чтобы запустить данный чарт, воспользуемся командой `helm install nginx nginx-chart-v1`

Можно проверить, что всё успешно запустилось, через следующие команды:

```bash
kubectl get svc
kubectl get deploy
kubectl get pods
```

Проверить доступность nginx сервера можно командой `kubectl port-forward service/nginx-service 8081:32080`. Затем нужно открыть браузер, например Google Chrome, в поисковой строке набрать localhost:8081 и нажать Enter. Страничка сервера появится.

## 3 Развернуть Jenkins в Minikube

Чтобы развернуть Jenkins, нужно воспользоваться следующими командами:

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
kubectl create namespace jenkins
helm install jenkins jenkins/jenkins --namespace jenkins
```

После выполнения данных команд появится сообщение:

```bash
NAME: jenkins
LAST DEPLOYED: Fri May  3 11:10:05 2024
NAMESPACE: jenkins
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
echo http://127.0.0.1:8080
kubectl --namespace jenkins port-forward svc/jenkins 8080:8080
3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://127.0.0.1:8080/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos
For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/
NOTE: Consider using a custom image with pre-installed plugins
```

Нужно подождать, пока все поды Jenkins будут готовы. Проверить STATUS можно командой `kubectl get pods -n jenkins`. Статус должен быть __Running__

Чтобы зайти на Jenkins через браузер, нужно сделать __port-forward__. Выполним команду в терминале `kubectl port-forward service/jenkins 8080:8080 -n jenkins`. Теперь можно зайти в браузер, ввести в поисковой строке localhost:8080, и появится страничка авторизации Jenkins.

Чтобы узнать пароль от учетной записи, нужно в терминале ввести команду `kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo`. Терминал выдаст пароль. В моем случае это __pTnQvZH4s8klClKzNeH8v3__

Теперь можно зайти в Jenkins:

Логин - admin

Пароль - pTnQvZH4s8klClKzNeH8v3

## 4 Написать Jenkins Job, который устанавливает написанный чарт для установки Nginx на кластер Minikube.

Чтобы создать Jenkins Job, нужно на главной странице:

1. Нажать на __Создать Item__
2. Ввести имя Item'a
3. Выбрать __Создать задачу со свободной конфигурацией__
4. Нажать __Ок__

Появится новая страничка __Общие настройки__

Здесь нужно в __Управление исходным кодом__ выбрать __Git__. В поле __Repository URL__ вставить ссылку на репозиторий Github, в моем случае это https://github.com/anmoskva96/jenkins-nginx-k8s.git. Затем в __Шаги сборки__
нажать на __Добавить шаг сборки__, выбрать __Выполнить команду shell__.

Теперь необходимо вписать команды, которые будет выполнять Jenkins:

```bash
echo "----------START JOB----------"

echo "----------INSTALL HELM----------"

curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
tar -zxvf helm.tar.gz
mkdir -p /home/jenkins/bin
mv linux-amd64/helm /home/jenkins/bin/helm
export PATH=$PATH:/home/jenkins/bin
helm version

echo "----------START HELM CHART----------"

helm install nginx nginx-chart-v1

```

> Пояснение: Когда Jenkins начинает выполнять Job, он создает Pod с контейнером, к котором нет helm. Поэтому необходимо установить helm, чтобы наш чарт смог успешно выполниться. Я написал команды для скачивания и установки инструмента helm

После этого нужно нажать __Сохранить__

## 5 В Git репозиторий добавить Helm чарт

Я нахожусь в директории __/home/lemuelge/github/jenkins-nginx-k8s__. Это директория, из которой я буду делать пуш Helm чарт.

Стркуктура каталогов и файлов следующая:

```bash
├── nginx-chart-v1
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── values.yaml
└── README.md

3 directories, 5 files
```

Для добавления Helm чарт я использую команды:

```bash
git add nginx-chart-v1
git commit -m "Helm chart"
git push origin master
```

## 6 В Git репозиторий добавить файл Readme.md, где описан процесс запуска Minikube с Jenkins и процесс создания Job для Nginx

Выполним push README.md аналогичко п.5:

```bash
git add README.md
git commit -m "README.md"
git push origin master
```
***

Выполнив выше указанные действия, Helm chart с помощью Jenkins запустился, но завершился с ошибкой:

```bash
+ echo ----------START JOB----------
----------START JOB----------
+ echo ----------INSTALL HELM----------
----------INSTALL HELM----------
+ curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
+ tar -zxvf helm.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
+ mkdir -p /home/jenkins/bin
+ mv linux-amd64/helm /home/jenkins/bin/helm
+ export PATH=/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/jenkins/bin
+ helm version
version.BuildInfo{Version:"v3.7.1", GitCommit:"1d11fcb5d3f3bf00dbe6fe31b8412839a96b3dc4", GitTreeState:"clean", GoVersion:"go1.16.9"}
+ echo ----------START HELM CHART----------
----------START HELM CHART----------
+ helm install nginx nginx-chart-v1
Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: could not get information about the resource: services "nginx-service" is forbidden: User "system:serviceaccount:jenkins:default" cannot get resource "services" in API group "" in the namespace "jenkins"
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

Насколько я понял, пользователь Jenkins имеет недостаточные права для выполнения операций в k8s

Решение данной проблемы я не смог найти