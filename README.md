### Create token for dashboard

```
microk8s kubectl create token -n kube-system default --duration=8544h
```
or
```
kubectl create token default --duration=50000s
```
or
```
microk8s kubectl describe secret -n kube-system microk8s-dashboard-token
```
### change to NodePort to access from localnetwork:
```
kubectl -n kube-system edit service kubernetes-dashboard
```

### find cluster hostname
- go inside a pod and run
```
host [ip-of-cluster]
```
or you can run another tool to do reverse dns lookup

- it seems that cluster hostname is 
```
[ip-of-cluster].kubernetes.default.svc.cluster.local
```

### edit coredns configmap
for example look at data section of configmap:
```
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        rewrite name prefix example.com 192-168-1-198.kubernetes.default.svc.cluster.local
        ready
        log . {
          class error
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4 
        cache 30
        loop
        reload
        loadbalance
    }
```

```
rewrite name prefix example.com [ip-of-cluster].kubernetes.default.svc.cluster.local
```
or regex to capture all the subdomains:
```
rewrite name regex (.*)(\.)?wieru\.com 192-168-0-6.kubernetes.default.svc.cluster.local
```

- [Some more explanation](https://therubyist.org/2021/02/20/cert-manager-nat-loopback-and-coredns/)
- check out [CoreDNS plugins doc](https://coredns.io/plugins/rewrite/).

### Create storage class gocd-sc first
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gocd-sc
  annotations:
    openebs.io/cas-type: local
    cas.openebs.io/config: |
      - name: StorageType
        value: hostpath
      - name: BasePath
        value: /data/gocd
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### Install gocd helm:

#### configure helm to work with microk8s kubectl
```
kubectl config view --raw > ~/.kube/config
```
#### install
```
helm install gocd gocd/gocd --namespace gocd --set server.persistence.storageClass=gocd-sc
```

```
server = "https://registry-1.docker.io"

[host."registry-1.docker.io"]
  capabilities = ["pull", "resolve"]
```

### Create secret to add to registry deployment
```
kubectl create -n default secret docker-registry regcred --docker-server=10.152.183.154:5000 --docker-username=admin --docker-password=admin --docker-email=admin@example.com
```
### Attach this secret to the deployment
```
containers:
...
...
imagePullSecrets:
      - name: regcred
```

### Double check:
- check image
- check yml format: imagePullSecrets is right under spec (not under containers)
- check that the secret and the pod(s) are in the same namespace.

### Create kubernetes serviceaccount
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
```

### Create kubeconfig file

```
kubectl config view --raw > ~/.kube/config
```

### Helm commands

```
helm install full-coral ./mychart
helm install --debug --dry-run goodly-guppy ./mychart
```

### Create base64 encoded basic secrets

```
echo "mega_secret_key" | base64
```

##### Decode 

```
echo "bWVnYV9zZWNyZXRfa2V5Cg==" | base64 -d
```

### Create namespace

```
kind: Namespace
apiVersion: v1
metadata:
  name: plweb
  labels:
    kubernetes.io/metadata.name: plweb
```

### Create Cert manager clusterissuer

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
#change to your email
    email: wieriekenshin@gmail.com
    privateKeySecretRef:
       name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: public
```

