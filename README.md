

# Basic Setup (Assumes Ubuntu 20.04 Server Install)
Username: sysadmin
```
sudo apt install nfs-kernel-server
```
node IP addresses: 192.168.10.30, 192.168.10.31, 192.168.10.32
cluster IP addresses: 192.168.10.50 ... 192.168.10.69
## Setup Static IP Address
```
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
## Install Micro Kubernetes (Microk8s) with Snap (tested with v1.20.7)
```
sudo snap install microk8s --classic
snap info microk8s
sudo usermod -a -G microk8s sysadmin
sudo chown -f -R sysadmin ~/.kube
echo "# Microk8s aliases"             >> ~/.bash_aliases
echo "alias mkctl='microk8s kubectl'" >> ~/.bash_aliases
echo "alias helm='microk8s helm3'"    >> ~/.bash_aliases
```
## Verify Firewall is Disabled
```
sudo ufw status
```
## Install Microk8s packages
```
micro8s status
microk8s enable dns dashboard storage helm3
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
(Assumes "alias helm = 'microk8s helm'" exists in ~/.bash_aliases)
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
## Setup Persistent Volume Storage
My setup has an NFS server at 192.168.10.2 with a share named "/cluster" which will be mounted on /mnt/cluster of all nodes.
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
```
Create a namespace in the kubernetes cluster for nextcloud
```
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
### Install Packages
```
```
## Setup Ingress and Load Balancer Services
```
microk8s enable metallb:192.168.10.50-192.168.10.69
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace kube-system --set defaultBackend.enabled=false
vi ingress-service.yaml
mkctl --namespace kube-system get services -o wide -w ingress-nginx-controller
```
Take note of the ExternalIP address which will be used later:
```
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
ingress-nginx-controller   LoadBalancer   10.152.183.11   192.168.10.50   80:31537/TCP,443:32753/TCP   21s   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
^C
```
Visit the URL http://<ExternalIP>, should receive an nginx "404" error, indicating the proxy server is up but no services are available behind the proxy.
## Install Certificate Manager Custom Resource Definitions
Staging generates a certificate but that certificate is untrusted in the browser and will generate a browser warning.  Production certificate works as expected but has more limited rate limits.  Use staging for testing.  (See https://letsencrypt.org/docs/staging-environment/ for more details.)
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
    email: changeme
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
    email: changeme
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
mkctl get services -n kube-system -o wide
mkctl get pods -n kube-system -l app.kubernetes.io/instance=cert-manager -o wide
```
### Enable port forwarding from your internet IP address to the ExternalIP noted above
### Generate test certificate
```
vi cert-manager-ingress-test.yaml
```
~/cert-manager-ingress.test.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "cert-manager-ingress-test"
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - changeme
    secretName: "changeme-staging-tls"
  rules:
  - host: changeme
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: "http"
              port:
                number: 80
```
```
mkctl apply -f cert-manager-ingress-test.yaml
mkctl get certificaterequest -o wide
mkctl get certificate -o wide
mkctl delete -f cert-manager-ingress-test.yaml
```
## Installing Nextcloud
Fetch the Nextcloud default configuration options and modify to suit your needs.
```
helm show values nextcloud/nextcloud >> nextcloud-values.yaml
cp nextcloud-values.yaml nextcloud-values.yaml.orig
vi nextcloud-values.yaml
```
Suggest at least the following changes to nextcloud-values.yaml:
```
64c64
<   host: nextcloud.kube.home
---
>   host: changeme
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
Install the Nextcloud package
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
    - changeme
    secretName: "nextcloud-prod-tls"
  rules:
  - host: changeme
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
Now test if everything worked by visiting your new Nextcloud site at https://nextcloud.<domain>.
## Remove Nextcloud If You Made a Mistake
```
mkctl delete -f nextcloud-ingress.yaml
helm uninstall nextcloud --namespace nextcloud
```
## Remove Nodes from the Cluster
```
microk8s leave
microk8s reset
rm -rf ~/.kube/*
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
