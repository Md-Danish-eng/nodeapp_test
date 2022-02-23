
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

# First create 2 one master and one worker for cluster instance

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

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCakNDQWU2Z0F3SUJBZ0lCQVRBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwdGFXNXAKYTNWaVpVTkJNQjRYRFRJeU1ESXhOREV5TVRrMU5sb1hEVE15TURJeE16RXlNVGsxTmxvd0ZURVRNQkVHQTFVRQpBeE1LYldsdWFXdDFZbVZEUVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTE9WCk9KdE80MDJVZVJldksya3JaMVVDUGMySXJPT0k1V0tDVVRQWUtMWDdhTVo3YzNmSHpDTjh4UU1tWHBwNHlEOE0KcDlOR2x1eUJGc0lvYkFmZTN4ZWhlT0pTR29RR0dENndYSmZCeE5XcW9mRjNZRjlHSHJHRnhTZjdNQXd2eEx5UQpGSDk5MjExL3pBcHVpUlhKb014MWRWYXVwWUFsZkVjQjdFdUw2SFNyMjArOFcrQnZZM1JHWU1NRkEyOElVYmRnCmZzWW1QUFJ5V2ltUngvY3NTaDZRYWdiQS95UnVDb282dS9XMDBxVEhNMXFUV0ZOcW02TUMyVmxUNmZhVDhyWlIKMU5qMC9JQWkxRkltSXVia3VjZ0Y2aVcyc1h3cVNRVDl0bkhyNDNBdDJULzVGUXFkcUVCQks5RC9vRnprWURrVAo4cFhoR0RoMnBLUnJ2VlRrUFFVQ0F3RUFBYU5oTUY4d0RnWURWUjBQQVFIL0JBUURBZ0trTUIwR0ExVWRKUVFXCk1CUUdDQ3NHQVFVRkJ3TUNCZ2dyQmdFRkJRY0RBVEFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQjBHQTFVZERnUVcKQkJUekllaHRDYjZheHk0NU05S0czcVp1K1o1cmtUQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFLdVFEMmpwUgpLWG5VdW0wQ1BxZ2FEZlZkWlcxdXhlNmJubE5LSWdCMlhBK1hEMzVwK3Z0N2g1UDBmb0xBUzVMU3RQZmt6N1h0Ci9DTUxETy9ocTEvc0VCcUxncnA5VmlkK3c4bDQ0TllEYTRIdURydDFSNjdwd3dqUXRZWTlIVmRKQ1AzTmdIS08KeWlqWStDRWErT21KRWZWYUdFblRWZ2dpaWx2VElBa0s4YTZhbkJlSk0vckNaSVhXejEvemZNSkE4aWVXbEVwKwpNdVFXTkVTQ1VMZlZIOHAza0pPd1dXd0RSUkUwM2VXVVhPNGNIYlZ2N1BzbzJqQkl6ejlzWDN4WlVUWFlZcWNuClN6eWJDdjhseENGd2JZNXlVVnV5SVl6ZU1saXUydkxyY3BMTGZRK0lnb3h6akR4L0V6MktyWlIyc0JWZFpjYzIKSGNkQ2QzMDMzRlZCemc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==

    extensions:
    - extension:
        last-update: Wed, 23 Feb 2022 09:05:09 IST
        provider: minikube.sigs.k8s.io
        version: v1.25.1
      name: cluster_info
    server: https://192.168.0.103:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Wed, 23 Feb 2022 09:05:09 IST
        provider: minikube.sigs.k8s.io
        version: v1.25.1
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lCQWpBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwdGFXNXAKYTNWaVpVTkJNQjRYRFRJeU1ESXhOREV5TVRrMU5sb1hEVEkxTURJeE5ERXlNVGsxTmxvd01URVhNQlVHQTFVRQpDaE1PYzNsemRHVnRPbTFoYzNSbGNuTXhGakFVQmdOVkJBTVREVzFwYm1scmRXSmxMWFZ6WlhJd2dnRWlNQTBHCkNTcUdTSWIzRFFFQkFRVUFBNElCRHdBd2dnRUtBb0lCQVFEbWJwTmkrYkZmSFVJMjhadzBlOUV1Tkh6U1FDSHkKSUxYWDY1bnF6eWtIalozd2NObmc0Ukp1bWdFM0ZTZndiTjR5QzB5WVgzSHlRQzE5RnZKeEdEQ011SVo5Wk5GYQpXUXJVZk94RWFRanNNQmZ0NmkxcTNiTWRxS2pEQytneTZtQ1NnTUd4SmlxWjJEYWRqcTZmazZ3MzZ4VFBlVUc5CjRPTSsrOThZeUhlT3NvSlEvVmlRdDhKNWVtSlc2ODVFOFdLL2VvTitjQkIzSFFKa204Q1NFNzdKY0NSck50RTQKbnlUdFB5eE9Dek1lUnZObnhoTFRjRVI5a25NT1JWY3NOaFJ5Vk1yZitaN2pjZmNxUDI0YUg0dzNWMHA2R2dCdwpvV2lIOHZFcWkyUkkyOU9IRHFxRTlEVUZqeXpmV1dHM2VqUGx6WXhRY0pmMkdWREJwd3FqL0JSVEFnTUJBQUdqCllEQmVNQTRHQTFVZER3RUIvd1FFQXdJRm9EQWRCZ05WSFNVRUZqQVVCZ2dyQmdFRkJRY0RBUVlJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JUekllaHRDYjZheHk0NU05S0czcVp1K1o1cgprVEFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBTERPVWZDRm5xeVUxekNobU9aVWdLTkNMM1N4SHZoOURHMGhKCkZjOWV0dEtjZFZaNEJteFJlSnFsRTByUUkrem9KU2k0b0JMazBrRzlCaGdiNzZQNHN3T0s2VTdmY05EUkpOVjkKQ1gyMHRsdHUwRG1WazdGK1ZxYjQxY3VldTZoMHJGeGVyTUw0aHZoZmlLdS91Zi94OEZSTUlzMXRkZHJSdVVESgpzbEJMZnFSRVNGcmlHS1l1SlhkeGF3cW5FeXhhbHQyeHRwRHZQb1ZjODVIaG1kL0tzTDh5bWFJeHdUS2wrQUdiCm5NOUxhbEVnUXJlZ0VkU1FkOFM3Nk82UVFEUmFWTGJOelE4RFpMeHhjVi8wTU8vTnA4ZS9DenRMYnJ5Q1ZJVjYKaCtRelV6VW9FSFhQZEpQV3NXYmdnRG9ZNFYzdEk3ekZ0SHJ4VFJQdmdCQ2JsMHJKVHc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==

    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb2dJQkFBS0NBUUVBNW02VFl2bXhYeDFDTnZHY05IdlJMalI4MGtBaDhpQzExK3VaNnM4cEI0MmQ4SERaCjRPRVNicG9CTnhVbjhHemVNZ3RNbUY5eDhrQXRmUmJ5Y1Jnd2pMaUdmV1RSV2xrSzFIenNSR2tJN0RBWDdlb3QKYXQyekhhaW93d3ZvTXVwZ2tvREJzU1lxbWRnMm5ZNnVuNU9zTitzVXozbEJ2ZURqUHZ2ZkdNaDNqcktDVVAxWQprTGZDZVhwaVZ1dk9SUEZpdjNxRGZuQVFkeDBDWkp2QWtoTyt5WEFrYXpiUk9KOGs3VDhzVGdzekhrYnpaOFlTCjAzQkVmWkp6RGtWWExEWVVjbFRLMy9tZTQzSDNLajl1R2grTU4xZEtlaG9BY0tGb2gvTHhLb3RrU052VGh3NnEKaFBRMUJZOHMzMWxodDNvejVjMk1VSENYOWhsUXdhY0tvL3dVVXdJREFRQUJBb0lCQUZEQ3BtTkU4ZFpORWR6aAoxd1pKOHVsSHVndVNNSk9FeFZhMG14QkJwTGFoK3AyL1g0MUNOTXlRcXlaY0F0QnZ4M3d3bTVxM3NON2ZnVkhiCkRnTjNIK1RoOHpqVmNjNUJjTnRDSVNoa3k1ekR5azgzQ00zd2Y0dEFoazA5eWhhMk1EeUlaZG9wYnpyV0hXWWgKRUxDYTkvdnRKekVENVhlZjd1VXZMMlNuTmNmTHZFQ2l1NzA5RFRqUlNVQ1UyTHZCK29vSmxEeGRCNStqRzVVMApLRUpCaEZFc2pSN1d3N1VPc3ZCUklnNFNFcDU0UktTdEcrNDc0eVJHMkZndTBxMEtFWVU3aC9wOTlIa1hZdHZXClFqMWdnS3ZKa01WUlUxQ2dqanc4Q2ZaMjVua2V3NHJGYjBJaE1GY1VXUm9jWk5jeUZpenVuM2doUWhIUjY5WkUKdWc0bVdia0NnWUVBOEdJdUU2dU9OMXMvSERjNFZSN2Z1Q1NwSVQ3UTFqYmg2RVhwNWhIVVdGVEtjWjRRZXV0VQpJTU1hYnJxc00vbnczU0xqK3NGZ05KN01YV1g2bWZyQUR3WllvOWJ1VUxvMkk5YzZiZ2RiN3BmQ0ZyOGpjY0h6CldmYmdLdE1ZaGgydGw1R2JPUlNTUHV3dTA4b2tsdnNUQXJtendoaXRCWkhxeXZZODU2TE50c2NDZ1lFQTlXYmsKRmF3VDNFSXJnM0tXdHVZaGQ2WEZ4ei9SMC92ZzkxMXB3bkhJaWViU29qWnJmUlEyYmo4MUoxVW4zUkJ4WVlZWgpGeEJLNnI0c1ozUWVRbFBGQTVGOUpjeEV5MEhWVWo4THFoY1VqYkY2MDJFUUVzbzdvTVp2U24reVRCNU9FY1JzCkhzY3ExNEI1cmZrQXIvM09NYnExaURTL0gyRm9GNzRDNy9XMU9oVUNnWUFUeFI5aEFzVUpqSG1lU25SWm05WnUKZ0tWZ1ZKZzhaZnNpYlUyVlhIWUlaY0RZbzFWYnBxc2VucTAzMmlaN2g5emxjdzhvK21wOUtXcEpiQysySmtkUgpkUVlwUTI0S09hWm1RRGRRQVU3d1NvN3Q2LzV3UnJGSy91RGs1TU9wbEJ0STBmTGdPTzdtT2VxSUJLSUp3TkNKCmN0aHo2QytpdTZPQjJjcWNpbWs4MVFLQmdDY0FvMStPYWRtbjZxS0pvOHFONk9QTFJSUFY0Tk9BUk5FTDE3TS8Kd2srb2ovR1lGSjFjaVFvY29hWU9zcmMvMWNWYU9zS2ZwRWlLMFNQZ0lLOEtBVlgvMlpRWVV4YTY3OXlTaUpnUAo4d1JTSU9OWG1lWmluZmQva2xDVTJ4R2QvMnB6Zlh1bXkvaFVRd0tUZ0xoMzdqMlpIeUQyd1NtTG9hK2tVM012CjZnM0JBb0dBV3dPdVV0MGtFTWx5UERyQ0VLaTZwR0lLQzUwSTdFR0Yrb08yakVUU2RzTTVnZVE5bWsxWHR2L3AKTzVrb004ekVRV3hPRGdWYlQrZ1duNFo0WmdSdE43ai85OEdmNUpiSjB6cEtYUXV2SFNvQytUZVdYQm9HTDdZTQpuMWhQM1A5UmROSVdtdUtwMThHSkpDR0NoN29ZNm81UWw3UzRDdXBWdTZKWUNWcnc2VkU9Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==

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


















