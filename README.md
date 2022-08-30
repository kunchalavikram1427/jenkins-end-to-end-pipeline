# jenkins-end-to-end-pipeline

# Steps
## Install Jenkins
```
helm pull jenkins/jenkins --untar
```
Install
```
helm install jenkins jenkins/ --set controller.servicePort=80 --set controller.serviceType=LoadBalancer
```
  
