# bitnode
Bitnode is a set of containers, manifests, and utils for running a core bitnode server.

## Local Dev Installation

### Install Pre-reqs
Install Docker:
https://www.docker.com/products/docker-desktop/

Install k3s/k3d:
OS specific guide: https://avilpage.com/2023/03/setup-k8s-anywhere-k3d.html

On Mac, you can get running quickly with:
```
brew install kubectl
brew install k3d

# create cluster w/ 3 servers, 2 agents
k3d cluster create demo -p "3306:30080@agent:0" --servers 3 --agents 2
# or...
k3d cluster create demo --port 3306:3306@loadbalancer --agents 2
```

### Dolt
Setup Dolt:
```
# create dolt example namespace
kubectl create namespace dolt-cluster-example

# create dolt admin secret
kubectl \
  -n dolt-cluster-example \
  create secret generic \
  dolt-credentials \
  --from-literal=admin-user=root \
  --from-literal=admin-password=password

# create dolt
kubectl apply -f dolt-manifest.yaml

configmap/dolt created
service/dolt-internal created
service/dolt created
service/dolt-ro created
statefulset.apps/dolt created
role.rbac.authorization.k8s.io/doltclusterctl created
serviceaccount/doltclusterctl created
rolebinding.rbac.authorization.k8s.io/doltclusterctl created

# verify dolt is running (this can take a few seconds)
kubectl get pods -n dolt-cluster-example -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP          NODE                NOMINATED NODE   READINESS GATES
dolt-0   1/1     Running   0          16m   10.42.4.4   k3d-demo-agent-0    <none>           <none>
dolt-1   1/1     Running   0          16m   10.42.1.4   k3d-demo-server-1   <none>           <none>
dolt-2   1/1     Running   0          16m   10.42.3.4   k3d-demo-agent-1    <none>           <none>

# apply primary labels
kubectl run -i --tty \
    -n dolt-cluster-example \
    --image public.ecr.aws/dolthub/doltclusterctl:latest \
    --image-pull-policy Always \
    --restart=Never \
    --rm \
    --override-type=strategic \
    --overrides '
{
  "apiVersion": "v1",
  "kind": "Pod",
  "spec": {
    "serviceAccountName": "doltclusterctl",
    "containers": [{
      "name": "doltclusterctl",
      "env": [{
        "name": "DOLT_USERNAME",
        "valueFrom": {
          "secretKeyRef": {
            "name": "dolt-credentials",
            "key": "admin-user"
          }
        }
      }, {
        "name": "DOLT_PASSWORD",
        "valueFrom": {
          "secretKeyRef": {
            "name": "dolt-credentials",
            "key": "admin-password"
          }
        }
      }]
    }]
  }
}
' \
    doltclusterctl -- \
    doltclusterctl -n dolt-cluster-example applyprimarylabels dolt

2025/05/26 04:50:23 running applyprimarylabels against dolt-cluster-example/dolt
2025/05/26 04:50:23 applied primary label to dolt-cluster-example/dolt-0
2025/05/26 04:50:23 applied standby label to dolt-cluster-example/dolt-1
2025/05/26 04:50:23 applied standby label to dolt-cluster-example/dolt-2
```

Setting up nginx (optional):
```
kubectl create deployment nginx --image=nginx
kubectl create service clusterip nginx --tcp=3306:3306
kubectl apply -f nginx.yaml
ingress.networking.k8s.io/nginx created
```


Afterwards things will look like this:
```
$ kubectl get pods --all-namespaces -o wide
NAMESPACE              NAME                                      READY   STATUS    RESTARTS      AGE   IP           NODE                NOMINATED NODE   READINESS GATES
default                nginx-676b6c5bbc-l7vgk                    1/1     Running   1 (49m ago)   57m   10.42.3.9    k3d-demo-agent-1    <none>           <none>
dolt-cluster-example   dolt-0                                    1/1     Running   1 (49m ago)   60m   10.42.3.10   k3d-demo-agent-1    <none>           <none>
dolt-cluster-example   dolt-1                                    1/1     Running   1 (49m ago)   60m   10.42.1.12   k3d-demo-agent-0    <none>           <none>
dolt-cluster-example   dolt-2                                    1/1     Running   0             49m   10.42.0.13   k3d-demo-server-0   <none>           <none>
kube-system            coredns-ccb96694c-x6mqg                   1/1     Running   0             49m   10.42.1.10   k3d-demo-agent-0    <none>           <none>
kube-system            local-path-provisioner-5cf85fd84d-mq7bd   1/1     Running   0             49m   10.42.3.8    k3d-demo-agent-1    <none>           <none>
kube-system            metrics-server-5985cbc9d7-ncnvb           1/1     Running   0             49m   10.42.1.9    k3d-demo-agent-0    <none>           <none>
kube-system            svclb-dolt-lb-2706416b-fgbzk              1/1     Running   1 (49m ago)   60m   10.42.0.11   k3d-demo-server-0   <none>           <none>
kube-system            svclb-dolt-lb-2706416b-jnqls              1/1     Running   1 (49m ago)   60m   10.42.3.12   k3d-demo-agent-1    <none>           <none>
kube-system            svclb-dolt-lb-2706416b-pcbwm              1/1     Running   1 (49m ago)   60m   10.42.1.8    k3d-demo-agent-0    <none>           <none>
kube-system            svclb-traefik-9c1c052a-s9mhr              2/2     Running   2 (49m ago)   61m   10.42.1.7    k3d-demo-agent-0    <none>           <none>
kube-system            svclb-traefik-9c1c052a-tgd8m              2/2     Running   2 (49m ago)   61m   10.42.3.11   k3d-demo-agent-1    <none>           <none>
kube-system            svclb-traefik-9c1c052a-zd7fr              2/2     Running   2 (49m ago)   61m   10.42.0.12   k3d-demo-server-0   <none>           <none>
kube-system            traefik-5d45fc8cc9-krsns                  1/1     Running   1 (49m ago)   61m   10.42.1.11   k3d-demo-agent-0    <none>           <none>

$ kubectl get services -n dolt-cluster-example
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                        PORT(S)          AGE
dolt            ClusterIP      10.43.254.3     <none>                             3306/TCP         118s
dolt-internal   ClusterIP      None            <none>                             3306/TCP         118s
dolt-lb         LoadBalancer   10.43.189.134   172.18.0.3,172.18.0.4,172.18.0.5   3306:30557/TCP   118s
dolt-ro         ClusterIP      10.43.107.77    <none>                             3306/TCP         118s
```

Connect:
```
kubectl port-forward svc/dolt -n dolt-cluster-example 3306:3306
mysql -u root -P 3306 -h 127.0.0.1 -p
```

Or you can use dolt-workbench:
```
docker pull dolthub/dolt-workbench:latest
docker run -p 9002:9002 -p 3000:3000 dolthub/dolt-workbench:latest
```

Then you can connect in the UI on port 3000...make sure to port-forward like above. Here are some screenshots:
<img width="706" alt="Screenshot 2025-05-26 at 12 22 45 PM" src="https://github.com/user-attachments/assets/88f4fbde-b4a4-449b-9221-72694acf2e67" />

<img width="703" alt="Screenshot 2025-05-26 at 12 23 02 PM" src="https://github.com/user-attachments/assets/6264b63e-22a1-4ac9-852f-1de5eb33d2bc" />

<img width="698" alt="Screenshot 2025-05-26 at 12 23 19 PM" src="https://github.com/user-attachments/assets/49bb9e18-4243-440d-8646-d557fc0a456f" />

<img width="1426" alt="Screenshot 2025-05-26 at 12 21 50 PM" src="https://github.com/user-attachments/assets/cfb79ecd-7ae9-42e1-8fbb-bae9a1ff1ccf" />


Useful links on Dolt:
https://gist.github.com/reltuk/8a97771e8b46dd32e47c80ef0c3645f7
https://github.com/dolthub/doltclusterctl/issues/1#issuecomment-1642440594
https://github.com/dolthub/doltclusterctl

Exposing ports in k3d:
https://k3d.io/v5.0.1/usage/exposing_services/
https://k3d.io/v5.0.1/usage/commands/k3d_cluster_edit/

Example of using k3d cluster edit post-facto to edit ports:
```
$ k3d cluster edit demo --port-add 3306:3306@agent:0
INFO[0000] Renaming existing node k3d-demo-serverlb to k3d-demo-serverlb-IehQf...
INFO[0000] Creating new node k3d-demo-serverlb...
INFO[0000] Stopping existing node k3d-demo-serverlb-IehQf...
INFO[0010] Starting new node k3d-demo-serverlb...
INFO[0010] Starting node 'k3d-demo-serverlb'
INFO[0016] Deleting old node k3d-demo-serverlb-IehQf...
INFO[0016] Successfully updated demo
```

