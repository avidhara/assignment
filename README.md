# Hello World Applicaiton

## Build
to build Docker Image Locally locally

```bash
docker build -t $TAG .
```

### Deploying to Kubernetes Cluster

For this application assuming you are using Docker-Desktop.

### Deploy Cert-manager
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
```
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.1 \
  --set crds.enabled=true
```

### Deploy Ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

Once all the pods are up and running in `ingress-nginx` namespace check the services 

```bash
kubectl get pods -n ingress-nginx        
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-59ff46fdb6-s8kzz   1/1     Running   0          3m2s

kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.108.16.240   localhost     80:30420/TCP,443:30922/TCP   3m10s
ingress-nginx-controller-admission   ClusterIP      10.98.67.173    <none>        443/TCP                      3m10s
```


### letsencrypt Configurations

create `letsencrypt-staging.yaml` file with below content

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: rajivreddy.j@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            ingressClassName: pomerium
```
Validate Issuer state
```bash
kubectl get Issuer -A                                                 
NAMESPACE      NAME                  READY   AGE
cert-manager   letsencrypt-staging   True    16s
```

### Deploy Hello World application

```bash
cd hello-world

helm install hello-world . -f override.yaml
NAME: hello-world
LAST DEPLOYED: Mon Jul 22 15:59:00 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  http://hello-world.local/
```

```bash
kubectl get pods,svc,ing         
NAME                              READY   STATUS    RESTARTS   AGE
pod/hello-world-c86946978-df798   1/1     Running   0          67s

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/hello-world   ClusterIP   10.100.37.59   <none>        80/TCP    67s
service/kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP   9m15s

NAME                                    CLASS   HOSTS               ADDRESS     PORTS   AGE
ingress.networking.k8s.io/hello-world   nginx   hello-world.local   localhost   80      67s
```

update your `/etc/hosts` file for accessing the application


## Monitoring and Logging

### Deploy kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
create monitoring-override.yaml with below content

```yaml
grafana:
  enabled: true
  persistence:
    enabled: true
    storageClassName: "default"
    accessModes:
      - ReadWriteOnce
    size: 2Gi
    finalizers:
      - kubernetes.io/pvc-protection
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
    - monitoring.local
    tls:
    - secretName: monitoring-tls
      hosts:
        - monitoring.local
prometheus:
  prometheusSpec:
    persistentVolumeClaimRetentionPolicy: 
      whenDeleted: Retain
    retention: 100d
    storageSpec:
    # Using PersistentVolumeClaim
    #
      volumeClaimTemplate:
        spec:
          storageClassName: default
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi
```

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f monitoring-override.yaml -n monitoring --create-namespace
NAME: kube-prometheus-stack
LAST DEPLOYED: Mon Jul 22 16:05:35 2024
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=kube-prometheus-stack"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```
```bash
kubectl get pods,svc,ing -n monitoring
NAME                                                            READY   STATUS             RESTARTS       AGE
pod/alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running            0              7m24s
pod/kube-prometheus-stack-grafana-7b57c47f4b-r8p8d              0/3     Pending            0              7m57s
pod/kube-prometheus-stack-kube-state-metrics-7c8d64d446-mrsrb   1/1     Running            0              7m57s
pod/kube-prometheus-stack-operator-85b765d6bc-rqcn6             1/1     Running            0              7m57s
pod/kube-prometheus-stack-prometheus-node-exporter-7smws        0/1     CrashLoopBackOff   6 (2m3s ago)   7m57s

NAME                                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                            ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   7m24s
service/kube-prometheus-stack-alertmanager               ClusterIP   10.96.168.139    <none>        9093/TCP,8080/TCP            7m57s
service/kube-prometheus-stack-grafana                    ClusterIP   10.101.57.205    <none>        80/TCP                       7m57s
service/kube-prometheus-stack-kube-state-metrics         ClusterIP   10.107.106.60    <none>        8080/TCP                     7m57s
service/kube-prometheus-stack-operator                   ClusterIP   10.111.225.40    <none>        443/TCP                      7m57s
service/kube-prometheus-stack-prometheus                 ClusterIP   10.101.215.196   <none>        9090/TCP,8080/TCP            7m57s
service/kube-prometheus-stack-prometheus-node-exporter   ClusterIP   10.110.253.178   <none>        9100/TCP                     7m57s

NAME                                                      CLASS   HOSTS              ADDRESS     PORTS     AGE
ingress.networking.k8s.io/kube-prometheus-stack-grafana   nginx   monitoring.local   localhost   80, 443   7m57s
```

Both Grafana and hello-world services can be accessible via ingress on port 443

