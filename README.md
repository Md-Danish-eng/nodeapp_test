
# Deploy sample nodejs app on k8s using cicd jenkins pipeline

prerequisite:

* Install nodejs
* Install jenkins
* Install minikube 

#Install nodejs:

curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -

sudo apt-get install -y nodejs

node --version && npm --version

npm init

npm i express --save

#Create a github repo and clone the repo in your machine

#create index.js file here

vi index.js

var express = require('express');
var app = express();

app.get('/', function (req, res) {
    res.send('{ "response": "Hello world from Danish" }');
});

app.get('/will', function (req, res) {
    res.send('{ "response": "Hello World" }');
});
app.get('/ready', function (req, res) {
    res.send('{ "response": " Great!, It works!" }');
});
app.listen(process.env.PORT || 3000);
module.exports = app;

:wq!

#create Dockerfile here

vi Dockerfile

FROM node:latest

WORKDIR /app

COPY package.json ./

RUN npm install

COPY . .

EXPOSE 3000
CMD [ "node", "index.js" ]

:wq!

#create deployment.yml file 

vi deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deployment
  labels:
    app: nodeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodeapp
  template:
    metadata:
      labels:
        app: nodeapp 
    spec:
      containers:
      - name: nodeserver
        image: mddanish123/nodeapp_test
        ports:
        - containerPort: 3000

        :wq!

#create deploymentservice.yml

apiVersion: v1
kind: Service
metadata:
  name: nodeapp-service
spec:
  selector:
    app: nodeapp 
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 3000
    nodePort: 31110

:wq!

#create .dockerignore

vi .dockerignore

.idea
node_modules
npm-debug.log
Jenkinsfile

:wq!

push all the files in your git repo

#Install minikube on machine min-2CPU, RAM-4GB 

sudo su

sudo apt update && apt -y install docker.io

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl

curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

apt install conntrack

 minikube start --vm-driver=none
 minikube status

#OR you can create k8s cluster

# First create 2 instance one master and one worker for cluster 

* for both master and worker:

apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet=1.15.4-00 kubeadm=1.15.4-00 kubectl=1.15.4-00 docker.io

#Setup the Kubernetes Master only

kubeadm init

To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#Join your nodes to your Kubernetes cluster

kubeadm join --token 702ff6.bc7aacff7aacab17 174.138.15.158:6443 --discovery-token-ca-cert-hash sha256:68bc22d2c631800fd358a6d7e3998e598deb2980ee613b3c2f1da8978960c8ab

It shows this output:

This node has joined the cluster:
* Certificate signing request was sent to master and a response was received.
* The Kubelet was informed of the new secure connection details.

#Now go to maste and run

kubectl get nodes

NAME      STATUS    ROLES     AGE       VERSION
master   Ready     master    8m        v1.9.3
worker   Ready     <none>    6m        v1.9.3

#Install jenkins on minikube

create jenkinsdeployment.yaml & jenkinsservice.yaml in minikube

vi jenkinsdeployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk11
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        emptyDir: { }
:wq!

vi jenkinsservice.yaml

apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: jenkins
:wq!

kubectl create namespace jenkins

kubectl create -f jenkinsdeployment.yaml -n jenkins

kubectl get deployments -n jenkins

kubectl get pods -n jenkins

kubectl logs <pod-name> -n jenkins

kubectl create -f jenkinsservice.yaml -n jenkins

kubectl get services -n jenkins

AME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
jenkins   NodePort   10.108.120.245   <none>        8080:30937/TCP   10m

minikube IP

copy the ip and port and hit in the chrome browser

you will get jenkins page 

then go to kubectl logs <pod-name> -n jenkins

and copy the password and paste 

then go with install suggested plugins 

enter the name and password and email and click next

jenkins dashboard is ready Now

go to manage jenkins > manage plugins > available

search Dockerpipeline & kubernetes Continuous Deploy

install and restart jenkins

#pipeline script:

pipeline {

  environment {
    dockerimagename = "mddanish123/nodeapp_test"
    dockerImage = ""
  }

  agent any

  stages {

    stage('Checkout Source') {
      steps {
        git 'https://github.com/Md-Danish-eng/nodeapp_test.git'
      }
    }

    stage('Build image') {
      steps{
        script {
          dockerImage = "sudo docker.build mddanish123/nodeapp_test"
        }
      }
    }

    stage('Deploying App to Kubernetes') {
      steps {
        script {
          kubernetesDeploy(configs:"deploymentservice.yml", kubeconfigId: "kubernetes")
        }
      }
    }

  }

}

Before this go to manage jenkins > credentias > global credentias > add credentias

add dockerhub username and password.

and choose kubernetes-config 

add kubernetes credentials here and certificate 

to get certificate go to vm and type

vi ~/.kube/config 

# and create certificate and then paste to jenkins

then build 

check console output SUCCESS

then go to minikube and type

kubectl get deployments 

NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
nodeapp-deployment   1/1     1            1           63m

kubectl get pods

NAME                                 READY   STATUS    RESTARTS   AGE
nodeapp-deployment-7987786d8-sk7c6   1/1     Running   0          12m

kubectl get svc

NAME              TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes        ClusterIP      10.96.0.1      <none>        443/TCP          5h2m
nodeapp-service   LoadBalancer   10.110.73.89   <pending>     5000:31110/TCP   63m

go to chrome and check ip:port

welcome to k8s nodejs from nodeapp-deployment-78d5c58cd-2rgsb!

THANKS!


















