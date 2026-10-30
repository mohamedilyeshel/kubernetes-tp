# Kubernetes deployment guide (Docker Desktop, Windows PowerShell)

This project contains three workloads: Angular frontend, Spring Boot backend, and MySQL database. Kubernetes runs them in the `tp-deploiement` namespace. The frontend and backend are reached through one NGINX Ingress; MySQL is internal only.

## 0. Prerequisites
Install Docker Desktop with Kubernetes enabled, and install `kubectl`. In Docker Desktop: Settings -> Kubernetes -> Enable Kubernetes. Wait for the cluster to show Running.

```powershell
kubectl config current-context
kubectl cluster-info
docker version
```

Use the `docker-desktop` context. Run every command below from the extracted `Tp_Docker_Kube/test-main` folder.

## 1. Install the Traefik Ingress controller (one time per cluster)
An Ingress resource only defines routing; it does not install the controller. This guide uses Traefik from its official Helm chart. The community ingress-nginx controller was retired in March 2026, so do not install it for a new project.

Install Helm 3, then run:

```powershell
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik --namespace traefik --create-namespace --set providers.kubernetesIngress.enabled=true --set ingressClass.enabled=true --set ingressClass.name=traefik --set ingressClass.isDefaultClass=true --set service.type=NodePort --set ports.web.nodePort=30080
kubectl get pods -n traefik
kubectl get service -n traefik
```

Wait until the Traefik pod is Ready. HTTP is exposed on NodePort 30080.

## 2. Set the database password
Edit `k8s/secret.yaml`. The `data.root-password` value must be Base64. Encode your chosen password in PowerShell with:
```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('YourStrongPassword'))
```
Replace the existing `root-password` value with the result. The current value is the Base64 encoding of `CHANGE_ME_BEFORE_DEPLOY`, so it is only a placeholder. The backend reads the same value from the Secret; the password is not stored in the ConfigMaps. This simple Secret manifest is for a classroom/local demo, not production secret management.

## 3. Build the two application images
```powershell
docker build -t tp-deploiement-backend:latest .\springboot\app
docker build -t tp-deploiement-frontend:latest .\angular-app
docker image ls
```
The MySQL image is pulled automatically from Docker Hub when the StatefulSet starts. Docker Desktop Kubernetes can use images built by the local Docker engine.

## 4. Deploy manifests (each Kubernetes resource has its own file)

Run these commands from the `test-main` folder, in this order:

```powershell
kubectl apply -f .\\k8s\\namespace.yaml
kubectl apply -f .\\k8s\\secret.yaml
kubectl apply -f .\\k8s\\configmap.yaml

kubectl apply -f .\\k8s\\mysql-service.yaml
kubectl apply -f .\\k8s\\mysql-pvc.yaml
kubectl apply -f .\\k8s\\mysql-statefulset.yaml

kubectl apply -f .\\k8s\\backend-service.yaml
kubectl apply -f .\\k8s\\backend-deployment.yaml

kubectl apply -f .\\k8s\\frontend-service.yaml
kubectl apply -f .\\k8s\\frontend-deployment.yaml

kubectl apply -f .\\k8s\\middleware.yaml
kubectl apply -f .\\k8s\\ingress.yaml
```

Or run the helper after setting the Secret value and building images:

```powershell
.\\scripts\\deploy-kubernetes.ps1
```

The middleware removes the `/api` prefix before sending the request to Spring Boot, whose controller uses `/user` as its base path. This makes the frontend URL `/api/user/...` reach the backend route `/user/...`.

## 5. Wait and verify
```powershell
kubectl get pods -n tp-deploiement -w
kubectl rollout status statefulset/mysql -n tp-deploiement
kubectl rollout status deployment/backend -n tp-deploiement
kubectl rollout status deployment/frontend -n tp-deploiement
kubectl get services,ingress,pvc -n tp-deploiement
kubectl describe ingress tp-deploiement-ingress -n tp-deploiement
```
All three pods should become `Running` and `READY 1/1`. The MySQL PVC should be `Bound`.

## 6. Open the application
Add this line to the Windows hosts file (`C:\Windows\System32\drivers\etc\hosts`, edit as Administrator):
```
127.0.0.1 app.local
```
Then open `http://app.local:30080` in your browser. Ingress sends `/` to Angular and `/api` to Spring Boot. The Angular app already calls `/api/user/...`, so both routes share the same host and no browser CORS configuration is needed.

If localhost does not reach the controller, inspect `kubectl get svc ingress-nginx-controller -n ingress-nginx` and use its EXTERNAL-IP/ports. You can also test routing with:
```powershell
curl.exe -H "Host: app.local" http://127.0.0.1:30080/
curl.exe -H "Host: app.local" http://127.0.0.1:30080/api/user/all
```

## 7. Useful troubleshooting commands
```powershell
kubectl get all,pvc -n tp-deploiement
kubectl logs statefulset/mysql -n tp-deploiement
kubectl logs deployment/backend -n tp-deploiement
kubectl logs deployment/frontend -n tp-deploiement
kubectl describe pod -n tp-deploiement
kubectl get events -n tp-deploiement --sort-by=.lastTimestamp
```
If a pod is `ImagePullBackOff`, check image names and rebuild. If backend restarts, inspect backend logs and confirm MySQL is Ready and the Secret password matches. If PVC stays Pending, check the cluster's default StorageClass with `kubectl get storageclass`.

## 8. Remove the application
```powershell
kubectl delete -f .\k8s\ingress.yaml
kubectl delete -f .\k8s\frontend.yaml
kubectl delete -f .\k8s\backend.yaml
kubectl delete -f .\k8s\mysql.yaml
kubectl delete -f .\k8s\configmap.yaml
kubectl delete -f .\k8s\secret.yaml
kubectl delete -f .\k8s\namespace.yaml
```
Deleting the PVC can delete the database volume depending on the StorageClass reclaim policy. Keep a backup if the data matters. The ingress-nginx controller is cluster-wide and is intentionally not removed by the application cleanup.

## Explanation for the presentation
- **Namespace**: isolates all resources belonging to this project.
- **ConfigMaps**: hold non-sensitive configuration such as database name, JDBC URL, username, and health-probe setting.
- **Secret**: stores the shared database password separately from ordinary configuration.
- **StatefulSet + PVC**: runs MySQL with stable identity and persistent storage so data survives a pod restart.
- **ClusterIP Services**: provide stable internal DNS names (`mysql`, `backend`, `frontend`) and hide changing pod IP addresses.
- **Deployments**: manage frontend/backend pods and allow Kubernetes to recreate failed pods.
- **Probes**: startup, readiness, and liveness checks control when containers are considered started, ready for traffic, or unhealthy.
- **Ingress**: one HTTP entry point routes `/api` to Spring Boot and `/` to Angular. Traefik implements those rules.
- **Image build**: Docker packages each app; Kubernetes schedules and runs those images.

## Architecture
Browser -> NGINX Ingress -> Angular frontend (`/`) or Spring Boot backend (`/api`)
Spring Boot backend -> MySQL Service (`mysql:3306`) -> PersistentVolumeClaim
