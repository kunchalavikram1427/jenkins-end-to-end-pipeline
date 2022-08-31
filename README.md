# Jenkins End to End Pipeline

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
or install without making any changes to the chart by
```
helm upgrade --install jenkins jenkins/jenkins --set controller.servicePort=80 --set controller.serviceType=LoadBalancer
```

## Install SonarQube
```
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
helm pull sonarqube/sonarqube --untar
```
Run
```
helm upgrade --install sonarqube sonarqube/ 
```

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

## PVC for Maven Cache
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
```
withSonarQubeEnv(credentialsId: 'sonar', installationName: 'sonarserver') { 
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
### Nexus Stage
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
### Wait for QG
```
timeout(time: 1, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}
```
### Helm Deployment
```
withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', serverUrl: '') {
    sh "helm upgrade --install petclinic petclinic-chart/"
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

## Plugins to use
```
https://plugins.jenkins.io/pipeline-stage-view/
https://plugins.jenkins.io/nexus-artifact-uploader/
https://plugins.jenkins.io/pipeline-utility-steps/
https://plugins.jenkins.io/junit/
https://plugins.jenkins.io/kubernetes-cli/
```

## Extras
Use these parameters if invoking Sonar Maven Plugin
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
