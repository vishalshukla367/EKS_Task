# EKS_Task

# Deploy Web App using EKS
# About EKS :
Amazon Elastic Kubernetes Service. Amazon Elastic Kubernetes Service (Amazon EKS) is a fully managed Kubernetes service. ... First, you can choose to run your EKS clusters using AWS Fargate, which is serverless compute for containers.








Add alt text
No alt text provided for this image
Kubernetes Cluster
A master node is a node which controls and manages a set of worker nodes (workloads runtime) and resembles a cluster in Kubernetes. A master node has the following components to help manage worker nodes: Kube-Controller-Manager, which runs a set of controllers for the running cluster.

The worker node(s) host the pods that are the components of the application workload. The control plane manages the worker nodes and the Pods in the cluster. In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability.








Add alt text
No alt text provided for this image
--> Launch our EKS cluster on AWS Cloud. I have launched cluster with with 2 nodes using yml file.

  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig

  metadata:
    name: ekscluster
    region:  ap-south-1

  nodeGroups:
     - name: nodeg1
       desiredCapacity: 2
       instanceType: t2.micro
       ssh:
           publicKeyName: clskey
    







Add alt text
No alt text provided for this image
--> EFS : Amazon Elastic File System is a cloud storage service provided by Amazon Web Services designed to provide scalable, elastic, concurrent with some restrictions, and encrypted file storage for use with both AWS cloud services and on-premises resources.








Add alt text
No alt text provided for this image
EFS provisioner with the help of YML file:

kind: Deployment
apiVersion: apps/v1
metadata:
  name: efs-provisioner
spec:
  selector:
    matchLabels:
      app: efs-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:v0.1.0
          env:
            - name: FILE_SYSTEM_ID
              value: fs-c2e66d13
            - name: AWS_REGION
              value: ap-south-1
            - name: PROVISIONER_NAME
              value: ekstask/aws-efs
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: fs-c2e45s13.efs.ap-south-1.amazonaws.com
            path: /


 Storage Class 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: mystorage
provisioner: kubernetes.io/aws-ebs
parameters: 
   type: st1
reclaimPolicy: Retain
PVC(Persistent Volume Claim): Dynamic provisioning of persistent volumes on AWS. A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator. It is a resource in the cluster just like a node is a cluster resource. A Persistent Volume Claim (PVC) is a request for storage by a user

Now Create PVC of 10 Gib

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: mystorage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
Launch Mysql Deployment ,create container of mysql image ,MySQL version 5.6 and get password of MySQL from secret (you have to create ) and mount our pvc to its path (var/lib/mysql). 



apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: passwordhere
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: efs-mysql


 Now Create wordpress , have loadbalance, anybody come to LB it connect elb, automatic redirect to port 80, get password form secrete 

apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: passwordhere
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: efs-wordpress


Now we have multiple file we want to run all file , use concept of kustomization 

Create Kustomization file :

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
secretGenerator:
- name: mysql-pass
  literals:
  - password=passwdhere 
resources:

  - mysql-deployment.yml
  - wordpress-deployment.yml
