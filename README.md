# Jenkins End to End Pipeline

## Youtube Video URL
```
https://youtu.be/l9mf8K3vBvQ
```
## Install Jenkins
```
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm pull jenkins/jenkins --untar
```
Run 
```
helm upgrade --install jenkins jenkins/
```
or install without making any changes to the chart by running
```
helm upgrade --install jenkins jenkins/jenkins --set controller.servicePort=80 --set controller.serviceType=LoadBalancer
```
Get password
```
kubectl exec -it jenkins-0 -- bash
cat /run/secrets/additional/chart-admin-password
```

## Install SonarQube
https://artifacthub.io/packages/helm/sonarqube/sonarqube
```
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
helm pull sonarqube/sonarqube --untar
```
Run
```
helm upgrade --install sonarqube sonarqube/ 
```
Password: admin

## Install Nexus
https://artifacthub.io/packages/helm/sonatype/nexus-repository-manager
```
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update
helm pull sonatype/nexus-repository-manager --untar
```
Run
```
helm upgrade --install nexus nexus-repository-manager/
```
Get Password
```
kubectl exec -it nexus-nexus-repository-manager-5745766765-5pjcf -- bash
cat /nexus-data/admin.password
```

## Install Mailhog
```
helm repo add codecentric https://codecentric.github.io/helm-charts
helm repo update
helm pull codecentric/mailhog --untar
helm upgrade --install mail mailhog/
```

## PVC for Maven Cache
https://kubernetes.io/docs/concepts/storage/persistent-volumes/
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: maven-cache
spec: 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Pipeline Steps Snippets

### Agent Template
https://plugins.jenkins.io/kubernetes/
```
kubernetes {
    yaml '''
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            app: test
        spec:
          containers:
          - name: maven
            image: maven:3.8.3-adoptopenjdk-11
            command:
            - cat
            tty: true
          - name: git
            image: bitnami/git:latest
            command:
            - cat
            tty: true
    '''
}
```
### Build Options
https://www.jenkins.io/doc/book/pipeline/syntax/#options
```
options {
      buildDiscarder logRotator(daysToKeepStr: '2', numToKeepStr: '10')
      timeout(time: 10, unit: 'MINUTES')
}
```
### Git clone
```
git url: 'https://github.com/kunchalavikram1427/spring-petclinic.git',
branch: 'main'
```
### Maven Build
```
mvn -Dmaven.test.failure.ignore=true clean package
```
### Build Image
```
docker build -t $IMAGE_NAME:$IMAGE_TAG .
docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
```
### Sonar Stage
https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/
https://www.jenkins.io/doc/pipeline/steps/sonar/
```
withSonarQubeEnv(credentialsId: 'CREDS', installationName: 'SONARSERVER') { 
    sh '''/opt/sonar-scanner/bin/sonar-scanner \
      -Dsonar.projectKey=petclinic \
      -Dsonar.projectName=petclinic \
      -Dsonar.projectVersion=1.0 \
      -Dsonar.sources=src/main \
      -Dsonar.tests=src/test \
      -Dsonar.java.binaries=target/classes  \
      -Dsonar.language=java \
      -Dsonar.sourceEncoding=UTF-8 \
      -Dsonar.java.libraries=target/classes
      '''
}

```
### Wait for QG
https://www.jenkins.io/doc/pipeline/steps/sonar/
```
timeout(time: 1, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}
```

### Nexus Stage
https://blog.sonatype.com/workflow-automation-publishing-artifacts-to-nexus-using-jenkins-pipelines
```
script {
    pom = readMavenPom file: "pom.xml";
    filesByGlob = findFiles(glob: "target/*.${pom.packaging}"); 
    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
    artifactPath = filesByGlob[0].path;
    artifactExists = fileExists artifactPath;
    if(artifactExists) {
        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
        nexusArtifactUploader(
            nexusVersion: NEXUS_VERSION,
            protocol: NEXUS_PROTOCOL,
            nexusUrl: NEXUS_URL,
            groupId: pom.groupId,
            version: pom.version,
            repository: NEXUS_REPOSITORY,
            credentialsId: NEXUS_CREDENTIAL_ID,
            artifacts: [
                [artifactId: pom.artifactId,
                classifier: '',
                file: artifactPath,
                type: pom.packaging],

                [artifactId: pom.artifactId,
                classifier: '',
                file: "pom.xml",
                type: "pom"]
            ]
        );

    } else {
        error "*** File: ${artifactPath}, could not be found";
    }
}
```

## Nexus Push via cURL
https://support.sonatype.com/hc/en-us/articles/115006744008-How-can-I-programmatically-upload-files-into-Nexus-3-
```
withCredentials([usernamePassword(credentialsId: 'CREDS', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
    sh "curl -v -u $USER:$PASS --upload-file target/${pom.artifactId}-${pom.version}.${pom.packaging} \
    http://NEXUS-SERVER-URL/repository/maven-hosted/org/springframework/samples/${pom.artifactId}/${pom.version}/${pom.artifactId}-${pom.version}.${pom.packaging}"
}
```


## Sending status to Github
https://docs.github.com/en/rest/commits/statuses#create-a-commit-status
Install Generic webhook trigger plugin
ACTION
$.action
HEAD
$.pull_request.head.ref
BASE
$.pull_request.base.ref
SHA_ID
$.pull_request.head.sha
Expression
^opened dev master$
Text
$action $head $base

```
withCredentials([string(credentialsId: 'TOKEN', variable: 'TOKEN')]) {
    sh "curl -u kunchalavikram1427:$TOKEN -X POST 'https://api.github.com/repos/kunchalavikram1427/spring-petclinic/statuses/$SHA_ID' -H 'Accept: application/vnd.github.v3+json' -d '{\"state\": \"success\",\"context\": \"Maven Build\", \"description\": \"Jenkins\", \"target_url\": \"$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/console\"}' "
}
```
In function form
```
void sendStatus(String stage, String status) {
    container('curl') {
        withCredentials([string(credentialsId: 'TOKEN', variable: 'TOKEN')]) {
            sh "curl -u kunchalavikram1427:$TOKEN -X POST 'https://api.github.com/repos/kunchalavikram1427/spring-petclinic/statuses/$SHA_ID' -H 'Accept: application/vnd.github.v3+json' -d '{\"state\": \"$status\",\"context\": \"$stage\", \"description\": \"Jenkins\", \"target_url\": \"$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/console\"}' "
        }
    }
}
```
Call function
```
sendStatus("Push to Nexus","success")
sendStatus("Deployment","failure")
```

### Helm Deployment
https://plugins.jenkins.io/kubernetes-cli/
```
withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'KUBECONFIG', namespace: '', serverUrl: '') {
    sh "helm upgrade --install petclinic petclinic-chart/"
}
```
## Post Build Actions
https://www.jenkins.io/doc/book/pipeline/syntax/#post
https://plugins.jenkins.io/mailer/
```
post {
    failure {
        mail to: 'vikram@gmail.com',
        from: 'jenkinsadmin@gmail.com',
        subject: "Jenkins pipeline has failed for job ${env.JOB_NAME}",
        body: "Check build logs at ${env.BUILD_URL}"
    }
    success {
        mail to: 'vikram@gmail.com',
        from: 'jenkinsadmin@gmail.com',
        subject: "Jenkins pipeline for job ${env.JOB_NAME} is completed successfully",
        body: "Check build logs at ${env.BUILD_URL}"
    }
}
```
## Dockerfile to build petclinic project docker image
```
FROM openjdk:8-jre-alpine
EXPOSE 8080
COPY target/*.war /usr/bin/spring-petclinic.war
ENTRYPOINT ["java","-jar","/usr/bin/spring-petclinic.war","--server.port=8080"]
```

## Dockerfile to build custom image with helm and kubectl cli tools
Image available in Dockerhub as: kunchalavikram/kubectl_helm_cli:latest
```
FROM alpine/helm
RUN curl -LO https://dl.k8s.io/release/v1.25.0/bin/linux/amd64/kubectl \
    && mv kubectl /bin/kubectl \
    && chmod a+x /bin/kubectl
 ```
## Kubernetes Deployment & Service files
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
  labels:
    app: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: petclinic
  template:
    metadata:
      labels:
        app: petclinic
    spec:
      containers:
      - name: petclinic
        image: kunchalavikram/spring-petclinic:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: petclinic
spec:
  type: LoadBalancer
  selector:
    app: petclinic
  ports:
    - port: 80
      targetPort: 8080
```

## Creating and using Helm hosted repository in Nexus

Upload Helm chart
```
helm package <folder-name>
curl -u admin:admin123 http://NEXUS-URL/repository/HELM-REPO-NAME/ --upload-file petclinic-chart-0.1.0.tgz -v
```

Adding Hosted Repo
```
helm repo add helm-hosted http://admin:admin123@NEXUS-URL/repository/HELM-REPO-NAME/
helm repo update
```
Fetching Charts from Nexus
```
helm repo update
helm pull helm-hosted/petclinic-chart
```

## Plugins to use
```
https://plugins.jenkins.io/pipeline-stage-view/
https://plugins.jenkins.io/nexus-artifact-uploader/
https://plugins.jenkins.io/pipeline-utility-steps/
https://plugins.jenkins.io/junit/
https://plugins.jenkins.io/kubernetes-cli/
https://plugins.jenkins.io/sonar/
https://plugins.jenkins.io/generic-webhook-trigger/
```

## Extras
Use these parameters if invoking Sonar Maven Plugin
```
-Dsonar.host.url=http://SONAR-URL
-Dsonar.login=sqp_dfe411bbe58dfba863f942c6b5fcac2f79e7db1a
```

## References
```
https://github.com/jenkinsci/kubernetes-plugin/tree/master/examples
https://help.sonatype.com/repomanager3/nexus-repository-administration/formats/docker-registry/ssl-and-repository-connector-configuration
https://support.sonatype.com/hc/en-us/articles/217542177?_ga=2.135356529.1307852621.1661838709-1983751057.1661838709
```

## Author
- Vikram K (www.youtube.com/c/devopsmadeeasy)
