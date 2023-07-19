## Introduction

This page is intended to provide steps to run Konga on Kubernetes cluster. Konga is an open source tool that enables you to manage your Kong API Gateway with ease.
It is developed by Panagis Tselentis a software engineer leaves in Amsterdam. I found this tool really handy for managing Kong APIs through UI. To know more about konga visit [github](https://github.com/pantsel/konga).

## Prerequisites

* Kubernetes Cluster 

You can provisioned the kubernetes cluster on any public cloud provider like [AWS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html), [Google](https://cloud.google.com/kubernetes-engine/docs/how-to), [Azure](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal), etc. Or you can provisioned self managed Kubernetes cluster.

* Basic understanding of Docker and Kubernetes

You will find many documentations/videos available online to grab the knowledge of Docker and Kubernetes.

* Database Servers to store its configuration (MySQL, PostgreSQL, MongoDB). I have used PostgreSQL as a backend you can find its documentation [here](./Postgres-Server.md).

#### To Deploy Konga on Kubernetes cluster, complete the following steps :

* Create Konga Namespace 
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: konga
``` 

* Create Secret to store PostgreSQL configuration.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-creds
  namespace: konga
type: Opaque
data:
  POSTGRES_DB: konga
  POSTGRES_USER: admin
  POSTGRES_PASSWORD: admin123
``` 

Note: The above to steps are already covered in PostgreSQL documentation, skip it if you already did above steps. 

* Deploy Konga

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: konga
  namespace: konga
spec:
  replicas: 1
  selector:
    matchLabels:
      app: konga
  template:
    metadata:
      labels:
        name: konga
        app: konga
    spec:
      nodeSelector:
        kubernetes.io/hostname: cluster-k-1-control-plane
      containers:
        - name: konga
          image: pantsel/konga
          env:
            - name: NODE_ENV
              value: production
          ports:
            - containerPort: 1337
      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: postgres-pv-claim
```
* Create Kubernetes Service object for Konga.
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: konga
  name: konga-svc
  namespace: konga
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 1337
  selector:
    app: konga
``` 
* Create Ingress rule to access Konga outside the Kubernetes Cluster. I did this deployment on Azure AKS with Azure Load Balancer, Azure Private DNS and Nginx Ingress Controller.
You can deploy it any public cloud just make sure you have prerequisites configured on your cluster. 

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: konga-ingress
  namespace: konga
  annotations:
    spec.ingressClassName: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
    - host: konga.localhost.serge
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: konga-svc
                port:
                  number: 80
  tls:
    - hosts:
        - konga.localhost.serge
      secretName: certificate-secret
``` 
Note: You need to change your DNS record name with konga.example.com and also the valid certificate secret object for your domain.

#####You can find all the above configuration in a single [konga.yaml](./konga.yaml) file.