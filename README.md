# jenkins-end-to-end-pipeline

# Steps to install

## Install Jenkins
```
helm repo add jenkins https://charts.jenkins.io
helm repo update
```
Install the helm chart
```
helm install jenkins jenkins/jenkins --set controller.servicePort=80 --set controller.serviceType=LoadBalancer
```
