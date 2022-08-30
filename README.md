# Jenkins End to End Pipeline

# Steps to install

## Install Jenkins
```
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm pull jenkins/jenkins --untar
```
Install 
```
helm upgrade --install jenkins jenkins/
```
or install without making any changes to the chart by
```
helm upgrade --install jenkins jenkins/jenkins --set controller.servicePort=80 --set controller.serviceType=LoadBalancer
```

## Install Nexus
https://artifacthub.io/packages/helm/sonatype/nexus-repository-manager
```
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update
helm pull sonatype/nexus-repository-manager --untar
```
Install
```
helm upgrade --install nexus nexus-repository-manager/
```

## Install SonarQube
```
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
helm pull sonarqube/sonarqube --untar
```
Install
```
helm upgrade --install sonarqube sonarqube/ 
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

## Dockerfile to build the image
```
FROM openjdk:8-jre-alpine
EXPOSE 8080
COPY target/*.war /usr/bin/spring-petclinic.war
ENTRYPOINT ["java","-jar","/usr/bin/spring-petclinic.war","--server.port=8080"]
```

## Kubernetes Deployment & Service files
```

```
## Plugins to use
```
https://plugins.jenkins.io/pipeline-stage-view/
https://plugins.jenkins.io/nexus-artifact-uploader/
https://plugins.jenkins.io/pipeline-utility-steps/
https://plugins.jenkins.io/junit/
```

## Fail Pipeline if Nexus Step Fails
By default, any failure in Nexus Upload step will not fail the pipeline. Instead pipeline will be marked as 'SUCCESS'
> Uploading: http://167.99.18.228:8081/repository/maven-hosted2/org/springframework/samples/spring-petclinic/15.99/spring-petclinic-15.99.pom

> Failed to deploy artifacts: Could not find artifact org.springframework.samples:spring-petclinic:war:15.99 in maven-hosted2 (http://167.99.18.228:8081/repository/maven-hosted2)

>Finished: SUCCESS

```
stage('Results') 
{
  steps 
    {
      script 
      {
        def log_output = currentBuild.rawBuild.getLog(10000);
        def result = log_output.find { it.contains('Failed to deploy artifacts') }
        if (result) 
        {
          error (result)
        }
      }
    }
}
```

## Scratch Space
```
-Dsonar.host.url=http://206.189.241.88:9000 \
-Dsonar.login=sqp_dfe411bbe58dfba863f942c6b5fcac2f79e7db1a
```

## References
```
https://github.com/jenkinsci/kubernetes-plugin/tree/master/examples
https://help.sonatype.com/repomanager3/nexus-repository-administration/formats/docker-registry/ssl-and-repository-connector-configuration
https://support.sonatype.com/hc/en-us/articles/217542177?_ga=2.135356529.1307852621.1661838709-1983751057.1661838709
https://stackoverflow.com/questions/67735377/nexus-artifact-upload-plugin-does-not-fail-pipeline-if-upload-fails
https://issues.jenkins.io/browse/JENKINS-38918
https://github.com/jenkinsci/nexus-artifact-uploader-plugin/pull/23/commits/7f0648dacaf7ff2dd0b4687b706a42071f527770
```

## Author
- Vikram K (www.youtube.com/c/devopsmadeeasy)
