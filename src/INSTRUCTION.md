Instructions for Testing and Accessing the Django ToDo Application in Kubernetes
This document provides step-by-step instructions for deploying and testing the Django ToDo application using Kubernetes.

# 1. Prerequisites 
- Docker installed,
- kubectl (Kubernetes command-line tool) installed.
- A working Kubernetes cluster (e.g., Minikube, Docker Desktop with Kubernetes, GKE, EKS, etc.).
# 2. Preparing the Docker Image
- Build the Docker image: open your terminal in the project's root directory and run:
```` docker login ````
```` docker build -t leoleiden/todoapp src ```` 
- Upload the image to a Docker registry or ensure your Kubernetes cluster can access local images:
````docker tag todolist-app:latest leoleiden/todolist-app:latest````
````docker push leoleiden/todolist-app:latest````

# 3. Create Kubernetes Manifests
- Create kubernetes-manifests folder:
````mkdir kubernetes-manifests````
- Create a file named kubernetes-manifests/pods.yaml with the following content:
```
apiVersion: v1
kind: Pod
metadata:
  name: todolist-pod-1
  labels:
    app: todolist
spec:
  containers:
  - name: todolist-container
    image: leoleiden/todoapp
    ports:
    - containerPort: 8080
    env:
    - name: DJANGO_SETTINGS_MODULE
      value: "todolist.settings"
    - name: PYTHONUNBUFFERED
      value: "1"
---
apiVersion: v1
kind: Pod
metadata:
  name: todolist-pod-2
  labels:
    app: todolist
spec:
  containers:
  - name: todolist-container
    image: leoleiden/todoapp
    ports:
    - containerPort: 8080
    env:
    - name: DJANGO_SETTINGS_MODULE
      value: "todolist.settings"
    - name: PYTHONUNBUFFERED
      value: "1"
```
- Create a file named kubernetes-manifests/clusterip-service.yaml with the following content:
```
apiVersion: v1
kind: Service
metadata:
  name: todolist-clusterip-service
spec:
  selector:
    app: todolist # Цей селектор співпадає з міткою на наших подах
  ports:
  - protocol: TCP
    port: 80 # Порт, який сервіс буде відкривати (внутрішній)
    targetPort: 8080 # Порт, на якому слухає контейнер (ваш Django додаток)
  type: ClusterIP
```
- Create a file named kubernetes-manifests/nodeport-service.yaml.yaml with the following content:
```
apiVersion: v1
kind: Service
metadata:
  name: todolist-nodeport-service
spec:
  selector:
    app: todolist # Цей селектор співпадає з міткою на наших подах
  ports:
  - protocol: TCP
    port: 80 # Внутрішній порт сервісу
    targetPort: 8080 # Порт, на якому слухає контейнер
    nodePort: 30080 # Порт, який буде відкрито на кожному вузлі кластера (повинен бути в діапазоні 30000-32767)
  type: NodePort
```
# 4. Apply Kubernetes Manifests
- Open your terminal in the directory where you saved the YAML files and run:
```
cd src/kubernetes-manifests/
kubectl apply -f pods.yaml
kubectl apply -f clusterip-service.yaml
kubectl apply -f nodeport-service.yaml
```
- Check the status of deployed resources:
```
kubectl get pods
kubectl get services
```
- You should see two pods (todolist-pod-1, todolist-pod-2) with Running status and two services (todolist-clusterip-service, todolist-nodeport-service).

# 5.Testing ClusterIP Service DNS from a BusyBox container
ClusterIP services are only accessible within the cluster. You can verify accessibility by running a temporary busybox container and making a request to the service.

Launch a temporary busybox pod:

```kubectl run -it --rm --restart=Never busybox --image=busybox -- /bin/sh```

You will land inside the busybox container shell.

Make a request to the ClusterIP service using its DNS name:
The DNS name for a service in Kubernetes follows the format <service-name>.<namespace>.svc.cluster.local. In our case, if you're using the default namespace, it will be todolist-clusterip-service.default.svc.cluster.local or simply todolist-clusterip-service.

Execute in the busybox shell:

```wget -O- todolist-clusterip-service```

Or, to check the Health Check API:

```wget -O- todolist-clusterip-service/api/health```

You should see the output Health OK, confirming that the application is accessible via the ClusterIP service.
Exit the busybox container:
```exit```

# 6. Accessing the ToDo Application using kubectl port-forward
``kubectl port-forward`` allows you to temporarily redirect traffic from your local machine to a pod or service port within the Kubernetes cluster.
Execute port forwarding for the ClusterIP service:

```kubectl port-forward service/todolist-clusterip-service 8000:80```

This command will redirect traffic from local port 8000 to port 80 of the todolist-clusterip-service.

Open a web browser and navigate to http://localhost:8000. You should see your Django ToDo application.

To stop port forwarding, press Ctrl+C in the terminal where the kubectl port-forward command is running.

# 7. Accessing the Application using NodePort Service
A NodePort service exposes the application on a static port (in our case, 30080) on each node of your Kubernetes cluster.

Find the IP address of your Kubernetes node.
You will need to get the IP address of one of your worker nodes. For example:

```kubectl get nodes -o wide```

Look for the EXTERNAL-IP or INTERNAL-IP of a node.

Open a web browser and navigate to:

```http://<NODE_IP_ADDRESS>:30080```

Replace <NODE_IP_ADDRESS> with the actual IP address of your node. You should see your Django ToDo application.
