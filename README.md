### jenkins设置

系统管理  > 系统设置 > Cloud > Kubernetes > Images > Kuberntests Pod Templates > 更改name 值是 maven 的镜像为 registry.elegame.com:5000/openshift3/jenkins-slave-maven-rhel7

### 切换项目
```
oc project demo
```

### 创建应用
```
oc new-app --image-stream=spring-boot-demo:latest --allow-missing-images
oc rollout pause dc/spring-boot-demo
oc set env dc spring-boot-demo JAVA_ARGS=--spring.config.location=/opt/conf/application.properties TZ=Asia/Shanghai LANG=zh_CN.UTF-8
oc rollout resume dc/spring-boot-demo
```

### 创建BuildConfig
```
oc new-build --name=spring-boot-demo --image-stream=java:8 --binary=true
```

### 创建pipeline

pipeline.yml
```
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  annotations:
    pipeline.alpha.openshift.io/uses: '[{"name": "jenkins", "namespace": "", "kind":
      "DeploymentConfig"}]'
  creationTimestamp: null
  labels:
    app: spring-boot-demo
    name: spring-boot-demo
  name: spring-boot-demo-pipeline
spec:
  failedBuildsHistoryLimit: 5
  nodeSelector: {}
  output: {}
  postCommit: {}
  resources: {}
  runPolicy: Serial
  source:
    type: None
  strategy:
    jenkinsPipelineStrategy:
      env:
      - name: DEV_PROJECT
        value: demo
      - name: STAGE_PROJECT
        value: demo-stage
      - name: ENABLE_QUAY
        value: "false"
      jenkinsfile: |-
        def mvnCmd = "mvn -s nexus-openshift-settings.xml"
        def APP_NAME = "spring-boot-demo"

        pipeline {
          agent {
            label 'maven'
          }
          stages {
            stage('Pull Code') {
              steps {
                git branch: 'master', url: 'https://github.com/wangxiaopeng65/spring-boot-demo.git'
              }
            }
            stage('Build App') {
              steps {
                sh "${mvnCmd}  clean package -DskipTests=true"
              }
            }
            stage('Code Analysis') {
              steps {
                script {
                  sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.cicd.svc:9000 -DskipTests=true"
                }
              }
            }
            stage('Test') {
              steps {
                sh "${mvnCmd} test"
                step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
              }
            }
            stage('Archive App') {
              steps {
                 sh "${mvnCmd} deploy -DskipTests=true  -DaltDeploymentRepository=nexus::default::http://nexus.cicd.svc.cluster.local:8081/repository/maven-snapshots"
              }
            }
            stage('Build Image') {
              steps {
                script {
                  APP_FILENAME = sh (
                    script: 'ls target/*.jar | awk -F "/" "{print \\$2}"',
                    returnStdout: true
                  ).trim()
                  openshift.withCluster() {
                    openshift.withProject("${DEV_PROJECT}") {
                      openshift.selector("bc", "${APP_NAME}").startBuild("--from-file=target/${APP_FILENAME}", "--wait=true")
                    }
                  }
                }
              }
            }
            stage('Deploy DEV') {
              steps {
                script {
                  openshift.withCluster() {
                    openshift.withProject("${DEV_PROJECT}") {
                      openshift.selector("dc", "${APP_NAME}").rollout().latest();
                    }
                  }
                }
              }
            }
            stage('Promote to STAGE?') {
              steps {
                timeout(time:15, unit:'MINUTES') {
                    input message: "Promote to STAGE?", ok: "Promote"
                }

                script {
                  openshift.withCluster() {
                      openshift.tag("${DEV_PROJECT}/${APP_NAME}:latest", "${STAGE_PROJECT}/${APP_NAME}:latest")
                  }
                }
              }
            }
            stage('Deploy STAGE') {
              steps {
                script {
                  openshift.withCluster() {
                    openshift.withProject("${STAGE_PROJECT}") {
                      openshift.selector("dc", "${APP_NAME}").rollout().latest();
                    }
                  }
                }
              }
            }
          }
        }
    type: JenkinsPipeline
  successfulBuildsHistoryLimit: 5
  triggers:
  - github:
      secret: Fr4SiALh
    type: GitHub
  - generic:
      secret: Fr4SiALh
    type: Generic

```
```
oc create -f pipeline.yml
```

### 运行pipeline

```
oc start-build spring-boot-demo-pipeline
```

### RedHat OpenJDK8 使用手册
https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html-single/red_hat_java_s2i_for_openshift/index
