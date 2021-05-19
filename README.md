

# Basic Setup (Assumes Ubuntu 20.04 Server Install)
Username: sysadmin
```
sudo apt install nfs-kernel-server
```
node IP addresses: 192.168.10.30, 192.168.10.31, 192.168.10.32
cluster IP addresses: 192.168.10.50 ... 192.168.10.69
## Setup Static IP Address
```
sudo vi 00-installer-config.yaml
sudo apt update
sudo apt upgrade
sudo vi /etc/netplan/00-installer-config.yaml
```
## Delete Cloud Init Package
```
sudo apt purge cloud-init
sudo rm -rf /etc/cloud && sudo rm -rf /var/lib/cloud/
sudo apt autoremove
```
## Install Micro Kubernetes (Microk8s) with Snap (v1.20.7)
```
sudo snap install microk8s --classic
snap info microk8s
sudo usermod -a -G microk8s sysadmin
sudo chown -f -R sysadmin ~/.kube
```
## Verify Firewall is Disabled
```
sudo ufw status
```
## Install Microk8s packages
```
micro8s status
microk8s enable dns dashboard storage helm3 metallb
micro8s status
```
## Review Cluster Status (Optional)
```
mkctl get pods
mkctl get node
mkctl get all --all-namespaces
```
## Add Additional Nodes to the Cluster
```
microk8s add-node
```
## Install Required Helm Repos
(Uses alias helm = "microk8s helm")
```
helm repo add stable https://charts.helm.sh/stable
```
### Nextcloud
```
helm repo add nextcloud https://nextcloud.github.io/helm/
```
### Nginx Ingress Controller
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
### Certificate Manager
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```
### Install Packages
```
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace kube-system --set defaultBackend.enabled=false
helm show values nextcloud/nextcloud >> nextcloud-values.yaml
helm show values nextcloud/nextcloud >> nextcloud-values.yaml.orig
helm install nextcloud nextcloud/nextcloud --namespace nextcloud --values nextcloud-values.yaml
helm install cert-manager jetstack/cert-manager --namespace kube-system
```
## Setup Persistent Volume Storage
Assumes NFS server is available and is mounted on all nodes in the cluster
```
sudo mkdir /mnt/cluster
sudo chown -R sysadmin:sysadmin /mnt/cluster
sudo vi /etc/fstab
```
/etc/fstab
```
192.168.10.2:/cluster   /mnt/cluster            nfs     rw      0       0
```
Mount the NFS share
```
sudo mount /mnt/cluster
mkdir /mnt/cluster/nextcloud
mkctl create namespace nextcloud
vi nextcloud-persistent-volume.yaml
```
~/nextcloud-persistent-volume.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "nextcloud-storage"
  labels:
    type: "local"
spec:
  storageClassName: "manual"
  capacity:
    storage: "500Gi"
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/cluster/nextcloud"
```
```
vi nextcloud-persistent-volume-claim.yaml
```
~/nextcloud-persistent-volume-claim.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "nextcloud"
  name: "nextcloud-storage"
spec:
  storageClassName: "manual"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "500Gi"
```
```
mkctl apply -f nextcloud-persistent-volume.yaml
mkctl get pv
mkctl apply -f nextcloud-persistent-volume-claim.yaml
mkctl get pvc -n nextcloud
```
# Recover From Mistakes Creating Persistent Volume
```
mkctl delete -f nextcloud-persistent-volume-claim.yaml
mkctl delete -f nextcloud-persistent-volume.yaml
```
## Setup Ingress Controller Service
```
microk8s enable ingress metallb:192.168.10.50-192.168.10.69
vi ingress-service.yaml
```
~/ingress-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: ingress
  namespace: ingress
spec:
  selector:
    name: nginx-ingress-microk8s
  type: LoadBalancer
  # loadBalancerIP is optional. MetalLB will automatically allocate an IP from its pool if not
  # specified. You can also specify one manually.
  # loadBalancerIP: x.y.z.a
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
```
```
mkctl apply -f ingress-service.yaml
mkctl -n ingress get svc
mkctl --namespace kube-system get services -o wide -w ingress-nginx-controller
```
## Install Certificate Manager Custom Resource Definitions
```
mkctl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace kube-system
mkctl get pods -n kube-system -l app.kubernetes.io/instance=cert-manager -o wide
vi letsencrypt-staging.yaml
```
~/letsencrypt-staging.yaml
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: dave@drc.nz
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
```
```
vi letsencrypt-prod.yaml
```
~/letsencrypt-prod.yaml
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: dave@drc.nz
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```
```
mkctl apply -f letsencrypt-staging.yaml
mkctl apply -f letsencrypt-prod.yaml
mkctl get services -n kube-system -l app=ingress-nginx -o wide
mkctl get services -n kube-system -o wide
vi nextcloud-values.yaml
```
```
64c64
<   host: nextcloud.kube.home
---
>   host: nextcloud.drc.nz
66c66
<   password: changeme
---
>   password: "********"
311c311
<   enabled: false
---
>   enabled: true
326a327
>   existingClaim: "nextcloud-storage"
328c329
<   size: 8Gi
---
>   size: 500Gi
```
```
helm install nextcloud nextcloud/nextcloud --namespace nextcloud --values nextcloud-values.yaml
mkctl get pods -n nextcloud
kubectl get services -n nextcloud -o wide
mkctl get services -n nextcloud -o wide
vi nextcloud-ingress.yaml
```
~/nextcloud-ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: "nextcloud"
  name: "nextcloud-ingress"
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  tls:
  - hosts:
    - "nextcloud.drc.nz"
    secretName: "nextcloud-prod-tls"
  rules:
  - host: "nextcloud.drc.nz"
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: "nextcloud"
              port:
                number: 8080
```
```
mkctl apply -f nextcloud-ingress.yaml
mkctl get certificaterequest -n nextcloud -o wide
mkctl get certificate -n nextcloud -o wide
```
## Remove Nextcloud If You Made a Mistake
```
mkctl delete -f nextcloud-ingress.yaml
helm uninstall nextcloud --namespace nextcloud
```
## Remove Nodes from the Cluster
```
microk8s leave
microk8s reset
```
## Nginx Ingress
https://kubernetes.github.io/ingress-nginx/deploy/#using-helm
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```
## Nextcloud
```
helm repo add nextcloud https://nextcloud.github.io/helm/
helm repo update
helm install nextcloud nextcloud/nextcloud
```
# Useful Links
https://blog.true-kubernetes.com/self-host-nextcloud-using-kubernetes/
https://kubernetes.io/docs/concepts/services-networking/ingress/
https://cert-manager.io/docs/tutorials/acme/http-validation/
https://gist.github.com/djjudas21/ca27aab44231bdebb0e72d30e00553ff
https://microk8s.io/docs/commands
https://greg.jeanmart.me/2020/04/13/build-your-very-own-self-hosting-platform-wi/
