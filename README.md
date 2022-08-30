# jenkins-end-to-end-pipeline

# Steps
## Install Jenkins
```
helm pull jenkins/jenkins --untar
```
Change service type to loadbalancer & service port from 8080 to 80 in values.yaml file
