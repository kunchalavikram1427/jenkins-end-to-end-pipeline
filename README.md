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
https://artifacthub.io/packages/helm/sonatype/nexus-repository-manager
```
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update
```
Install
```
helm pull sonatype/nexus-repository-manager --untar
```

## PVC for Maven Cache
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: maven-repo
spec: 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

# References
```
https://github.com/jenkinsci/kubernetes-plugin/tree/master/examples
```
