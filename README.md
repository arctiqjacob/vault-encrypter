# Encrypter

This application is used to compare two secret engines, Transit and Transform, in HashiCorp Vault.

## Build and Push the Docker Image
Use the following steps to build the docker image and push it to Docker Hub.
```shell
# Build the Docker image
~]$ sudo docker build --tag encrypter .
...
Successfully tagged encrypter:latest

# Tag the Docker image with the latest tag
~]$ sudo docker tag encrypter jacobmammoliti/encrypter

# Push the Docker image to Docker Hub
~]$ sudo docker push jacobmammoliti/encrypter:latest
...
latest: digest: sha256:xxx
```

## Build and Deploy Kubernetes Manifest
Use the following steps to deploy the application into Kubernetes.
``` shell
# Create PVC to persist encrypted data.
~]$ kubectl apply -f <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: encrypter-pv-claim
  labels:
    app: encrypter
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Create Kubernetes secret for Vault Approle SECRET_ID
~]$ kubectl create secret generic vault-secret-id --from-literal=id=XXX

# Create deployment. Be sure to update the VAULT_ADDR and ROLE_ID.
~]$ kubectl apply -f <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: encrypter-deployment
  labels:
    app: encrypter
spec:
  selector:
    matchLabels:
      app: encrypter
  template:
    metadata:
      labels:
        app: encrypter
    spec:
      containers:
      - name: encrypter
        image: jacobmammoliti/encrypter:latest
        ports:
        - containerPort: 8080
        env:
        - name: VAULT_ADDR
          value: ""
        - name: ROLE_ID
          value: ""
        - name: SECRET_ID
          valueFrom:
            secretKeyRef:
              name: vault-secret-id
              key: id
        volumeMounts:
        - name: data
          mountPath: /mnt/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: encrypter-pv-claim
EOF

# Create a LoadBalancer type service to expose the deployment.
# Use ClusterIP if planning to use an Ingress Controller.
~]$ kubectl apply -f <<EOF
apiVersion: v1
kind: Service
metadata:
  name: encrypter-service
spec:
  selector:
    app: encrypter
  ports:
    - port: 8080
      targetPort: 8080
  type: LoadBalancer
EOF
```