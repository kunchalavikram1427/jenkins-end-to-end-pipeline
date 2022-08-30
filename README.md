# jenkins-end-to-end-pipeline

# Steps to install

## Install Jenkins
```
helm pull jenkins/jenkins --untar
```
Install the helm chart
```
helm install jenkins jenkins/jenkins --set controller.servicePort=80 --set controller.serviceType=LoadBalancer
```
