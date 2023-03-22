# IndellientREADme
## K8s manifest file, Dockerfile and Jenkins pipeline-as-code for a sample spring-book application deployment with mongoDB.
# ConfigMap for ecommerceapp
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecommerceconfigmap
data:
  mongousername: devdb
  mongopassword: devdb@123

---
# Deployment for mongoDB app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommercedeployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ecommerceapp
  template:
    metadata:
      name: ecommercepod
      labels:
        app: ecommerceapp
    spec:
      containers:
      - name: ecommercecontainer
        image: kalaski/spring-boot-mongo
        ports:
        - containerPort: 8080
        env:
        - name: MONGO_DB_USERNAME
          valueFrom:
            configMapKeyRef:
              name: ecommerceconfigmap
              key: mongousername
        - name: MONGO_DB_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: ecommerceconfigmap
              key: mongopassword
---
apiVersion: v1
kind: Service
metadata:
  name: ecommerceappsvc
spec:
  selector:
    app: ecommerceapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodbpvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: mongodbrs
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
     metadata:
       name: mongodbpod
       labels:
         app: mongodb
     spec:
       volumes:
       - name: pvc
         persistentVolumeClaim:
           claimName: mongodbpvc
       containers:
       - name: mongodbcontainer
         image: mongo
         ports:
         - containerPort: 27017
         env:
         - name: MONGO_INITDB_ROOT_USERNAME
           valueFrom:
             configMapKeyRef:
               name: ecommerceconfigmap
               key: mongousername
         - name: MONGO_INITDB_ROOT_PASSWORD
           valueFrom:
             configMapKeyRef:
               name: ecommerceconfigmap
               key: mongopassword
         volumeMounts:
         - name: pvc
           mountPath: /data/db
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
=====================================================================================================================================================
# Dockerfile used to build the image for the containerization of the ecommerce application
FROM openjdk:8-alpine
# Required for starting application up.
RUN apk update && apk add /bin/sh
RUN mkdir -p /opt/app
ENV PROJECT_HOME /opt/app
COPY target/spring-boot-mongo-1.0.jar $PROJECT_HOME/spring-boot-mongo.jar
WORKDIR $PROJECT_HOME
EXPOSE 8080
CMD ["java" ,"-jar","./spring-boot-mongo.jar"]

=========================================================================================================================================================
# Jenkins pipeline-as-code for application deployment
node{
	def mavenHome = tool name: "maven3.8.6"
	stage('CloneCode'){
	git 'https://github.com/Kalagbor/maven-web-app-mit'
	}
	stage('Test&Build'){
	sh "${mavenHome}/bin/mvn clean package"
	}
	stage('CodeQualityAnalysis'){
	//sh "${mavenHome}/bin/mvn sonar:sonar"
	}
	stage('UploadtoArtifactory'){
	//sh "${mavenHome}/bin/mvn deploy"
	}
	stage('DockerBuild'){
	sh "sudo docker build -t kalaski/ecommerce ." 
	}
	stage('6DockerPushtoDHR'){
	withCredentials([string(credentialsId: 'DockerHubCredential', variable: 'DockerHubCredential')]) {
    sh "sudo docker login -u kalaski -p ${DockerHubCredential}"  }
    sh "sudo docker push kalaski/mavenmit"
	}
	stage('7DeleteImage'){
    sh "docker rmi $(docker image -q)"
	}
	stage('DeployEcommerceApptoK8s'){
    sh "kubectl apply -f springapp.yml"
	}
}

