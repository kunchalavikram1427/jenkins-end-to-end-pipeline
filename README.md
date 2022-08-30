# jenkins-end-to-end-pipeline

# Steps to install

## Install Jenkins
```
helm repo add jenkins https://charts.jenkins.io
helm repo update
```
Install 
```
helm install jenkins jenkins/jenkins --set controller.servicePort=80 --set controller.serviceType=LoadBalancer
```

## Install Nexus
```
helm repo add curiedfcharts https://curie-data-factory.github.io/helm-charts
helm repo update
```
Install
```
helm install nexus curiedfcharts/nexus --set service.type=LoadBalancer
```


# References
```
https://github.com/jenkinsci/kubernetes-plugin/tree/master/examples
```
