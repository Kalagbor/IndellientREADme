# IndellientREADme
## K8s manifest file, Dockerfile and Jenkins pipeline-as-code for a sample spring-book application deployment with mongoDB.

apiVersion: v1
kind: ConfigMap
metadata: 
  name: springappconfigmap
data: 
  mongousername: admin
  mongopassword: admin123
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springappdeployment
spec:
  replicas: 2
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 1
  minReadySeconds: 30
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      name: springapppod
      labels:
        app: springapp
    spec:
      containers:
      - name: springappcontainer # This springappcontainer writes on the mongoDB container using the service springappsvc
        image: kalaski/spring-boot-mongo
        ports:
        - containerPort: 8080
        env:
        - name: MONGO_DB_USERNAME
          valueFrom:
            configMapKeyRef:
              name: springappconfigmap 
              key: mongousername
        - name: MONGO_DB_HOSTNAME
          valueFrom:
            configMapKeyRef: 
              name: springappconfigmap 
              key: mongopassword
        resources: 
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 1Gi
            
---
apiVersion: v1
kind: Service
metadata:
  name: springappsvc
spec:
  selector:
    app: springapp
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
           value: admin
         - name: MONGO_INITDB_ROOT_PASSWORD
           value: admin123
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
 ---
 apiVersion: autoscaling/v2beta1
 kind: HorizontalPodAutoscaler
 # The HPA autoscales the pod when the average CPU utilization gets to 70%
 # The HPA will the information of the CPU  utilization when Metric server is configured in the server.
 # The HPA scales the deployment with the tag springappdeployment
 metadata:
   name: springappdeployment
 spec:
   scaleTargetRef:
     apiVersion: apps/v1
     kind: Deployment
   miniReplicas: 2
   maxreplicas: 10
   metrics:
     - resource:
         name: cpu
         targetAverageUtilization: 70
       type: Resource
 
