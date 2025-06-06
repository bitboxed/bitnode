# bitnode

Bitboxed is building better roads and bridges from web2 and traditional public cloud to a decentralized, permissionless world.

Bitnode is a core set of containers, kubernetes manifests, and utils for running the core bitnode server with various decentralized platform services: storage, compute, networking, databases, and other cloud services. These services are being developed with decentralized infrastructure and public mining & consumption in mind, and therefore will remain free and open source to the public. All private infra or otherwise closed source derivative works should remain separate. 

## Local setup with k3s

We use [k3s](https://k3s.io/) because it closely emulates k8s, its lightweight and works well at the edge, and its easy to setup on any operating system.

### Local Dev Installation

Prereqs:
Clone the repo, and you may want to [install Docker](https://www.docker.com/products/docker-desktop/) if you haven't already.
```
git clone https://github.com/bitboxed/bitnode.git
cd bitnode
```

Setup multipass and k3s. There is a fantastic guide [here](https://dev.to/chillaranand/local-kubernetes-cluster-with-k3s-on-mac-m1-i57). Here's a list of steps i've used here:
```
brew install --cask multipass
multipass launch --name k3s --mem 2G --disk 10G

# mount your local bitnode directory on the VM
multipass mount ~/Github/bitnode k3s:~/bitnode

# shell in and install k3s
multipass shell k3s
ubuntu@k3s:~$ curl -sfL https://get.k3s.io | sh -

# open new terminal for k3s-worker node get token & ip of k3s for adding in the new node
multipass exec k3s sudo cat /var/lib/rancher/k3s/server/node-token
multipass info k3s | grep -i ip

# create a k3s-worker node in a new terminal and add it to the cluster (using your k3s ip and k3s token)
multipass launch --name k3s-worker --memory 2G --disk 10G
multipass shell k3s-worker
ubuntu@k3s-worker:~$ curl -sfL https://get.k3s.io | K3S_URL=https://192.168.64.4:6443 K3S_TOKEN="hs48af...947fh4::server:3tfkwjd...4jed73" sh -

# verify the k3 nodes are running now back on the k3s ternminal
ubuntu@k3s:~$ sudo kubectl get nodes -o wide
NAME         STATUS   ROLES                  AGE   VERSION        INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k3s          Ready    control-plane,master   35h   v1.32.5+k3s1   192.168.64.2   <none>        Ubuntu 24.04.2 LTS   6.8.0-60-generic   containerd://2.0.5-k3s1.32
k3s-worker   Ready    <none>                 35h   v1.32.5+k3s1   192.168.64.3   <none>        Ubuntu 24.04.2 LTS   6.8.0-60-generic   containerd://2.0.5-k3s1.32
```

### Dolt DB

[Dolt](https://github.com/dolthub/dolt) is a new SQL database that has a lot of powerful features, and enables rapid development andd collaboration with "git-like" features for databases. The [storage engine](https://docs.dolthub.com/architecture/storage-engine) also makes use of a novel prolly tree structure, which enables very fast diffs (highly useful for decentralized use cases).

If your node will be running Dolt, here's how you can setup it up quickly with [Direct-to-Standby Replication](https://docs.dolthub.com/sql-reference/server/replication#replication) on your k3s nodes.

```
# shell into k3s node and create dolt cluster example namespace
multipass shell k3s

sudo kubectl create namespace dolt-cluster-example
namespace/dolt-cluster-example created

# create dolt admin secret
sudo kubectl \
  -n dolt-cluster-example \
  create secret generic \
  dolt-credentials \
  --from-literal=admin-user=root \
  --from-literal=admin-password=password
secret/dolt-credentials created

# create dolt
sudo kubectl apply -f ~/bitnode/dolt-manifest.yaml

configmap/dolt created
service/dolt-internal created
service/dolt created
service/dolt-ro created
statefulset.apps/dolt created
role.rbac.authorization.k8s.io/doltclusterctl created
serviceaccount/doltclusterctl created
rolebinding.rbac.authorization.k8s.io/doltclusterctl created

# verify dolt is running (this can take a few seconds for all pods to come online)
sudo kubectl get pods -n dolt-cluster-example -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
dolt-0                           1/1     Running   0          18h   10.42.1.21   k3s-worker   <none>           <none>
dolt-1                           1/1     Running   0          18h   10.42.0.12   k3s          <none>           <none>
dolt-2                           1/1     Running   0          18h   10.42.1.20   k3s-worker   <none>           <none>
```

Next, we'll apply the Dolt `cluster_role=primary` label to the primary dolt pod, using [doltclusterctl](https://github.com/dolthub/doltclusterctl).
***note***: we're currently having issues w/ the arm64 version of this library, so we've compiled them and tagged them in our fork [here](https://github.com/bitboxed/doltclusterctl?tab=readme-ov-file#compilation-and-publishing-to-dockerhub).

```
sudo kubectl run -i --tty \
    -n dolt-cluster-example \
    --image priley86/doltclusterctl:arm64 \
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
' doltclusterctl -- -n dolt-cluster-example applyprimarylabels dolt

2025/06/05 21:59:18 running applyprimarylabels against dolt-cluster-example/dolt
2025/06/04 23:13:02 applied primary label to dolt-cluster-example/dolt-0
2025/06/04 23:13:02 applied standby label to dolt-cluster-example/dolt-1
2025/06/04 23:13:02 applied standby label to dolt-cluster-example/dolt-2
pod "doltclusterctl" deleted
```

Let's quickly verify we can connect to Dolt and things are running well on the cluster.
```
ubuntu@k3s:~$ sudo kubectl get services --all-namespaces
NAMESPACE              NAME             TYPE           CLUSTER-IP     EXTERNAL-IP                 PORT(S)                      AGE
default                kubernetes       ClusterIP      10.43.0.1      <none>                      443/TCP                      35h
dolt-cluster-example   dolt             ClusterIP      10.43.90.208   <none>                      3306/TCP                     20h
dolt-cluster-example   dolt-internal    ClusterIP      None           <none>                      3306/TCP                     20h
dolt-cluster-example   dolt-ro          ClusterIP      10.43.82.32    <none>                      3306/TCP                     20h
kube-system            kube-dns         ClusterIP      10.43.0.10     <none>                      53/UDP,53/TCP,9153/TCP       35h
kube-system            metrics-server   ClusterIP      10.43.125.24   <none>                      443/TCP                      35h
kube-system            traefik          LoadBalancer   10.43.150.35   192.168.64.2,192.168.64.3   80:31578/TCP,443:30956/TCP   35h

ubuntu@k3s:~$ sudo apt update
ubuntu@k3s:~$ sudo apt install -y mysql-client

# try logging in to the pod running dolt-0, using the local cluster ip address shown above for example
ubuntu@k3s:~$ mysql -u root -p -h 10.42.1.21 -D mysql --protocol=TCP
```

If you can now login w/ the sample user (`root`) and pw (`password`) created in the secret from before, now you're up and running with Dolt on k3s! :star:

#### dolt-workbench

[dolt-workbench](https://github.com/dolthub/dolt-workbench) is a very useful tool for visualizing your Dolt databases. Here's how you can run it and connect to it alongside your cluster.

Edit your `/etc/hosts` on your host system like below to match the domains used in our sample [dolt-workbench.yaml](dolt-workbench.yaml), and the internal ip address of your k3s-worker node (or retrieve it by running `multipass info k3s-worker | grep IPv4 | awk '{print $2}'`):
```
sudo vi /etc/hosts

 127.0.0.1   localhost
 192.168.64.3 app.dolt.test api.dolt.test
 ```


Next, apply the [dolt-workbench.yaml](dolt-workbench.yaml) on your k3s cluster:
```
multipass shell k3s
ubuntu@k3s:~$ sudo kubectl apply -f ~/bitnode/dolt-workbench.yaml

deployment.apps/dolt-workbench created
service/dolt-workbench created
ingress.networking.k8s.io/dolt-workbench-app-ingress created
ingress.networking.k8s.io/dolt-workbench-api-ingress created
```

Afterwards, you should now be able to view Dolt Workbench at `http://app.dolt.test/` and login to your dolt databases locally within the cluster using the admin user/pw secret mentioned before, e.g: `mysql://root:password@dolt:3306` (***note***: `dolt` is service name for our primary dolt database and can be used as the host local to the cluster).
<img width="1265" alt="Screenshot 2025-06-05 at 2 49 45 PM" src="https://github.com/user-attachments/assets/5ff61a16-9380-486e-98ac-a5c5cef6527a" />
<img width="1397" alt="Screenshot 2025-06-05 at 2 50 29 PM" src="https://github.com/user-attachments/assets/541f5b27-d45f-47d1-bc6e-44f347b1c52e" />
<img width="1397" alt="Screenshot 2025-06-05 at 2 50 35 PM" src="https://github.com/user-attachments/assets/0138c44a-c2bf-4a20-a8d0-4acf88c85fb8" />
<img width="1397" alt="Screenshot 2025-06-05 at 2 51 07 PM" src="https://github.com/user-attachments/assets/b5b63ebb-b7f2-4568-a1bc-44abb3a61a14" />


#### Troubleshooting connectivity
Sometimes Chrome on macOS does not respect `/etc/hosts` with multipass b/c macOS uses multilayered DNS resolution, so the following can also help ensure this works as expected using `dnsmasq`:
```
brew install dnsmasq
# Configure dnsmasq to route all dolt.test domains to your K3s node IP
echo "address=/.dolt.test/$(multipass info k3s-worker | grep IPv4 | awk '{print $2}')" > $(brew --prefix)/etc/dnsmasq.conf
sudo brew services start dnsmasq

# Create resolver config for dolt.test to use dnsmasq
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee /etc/resolver/dolt.test > /dev/null
```

If your k3s ip changes, you can also do:
```
echo "address=/.dolt.test/NEW_IP" > "$(brew --prefix)/etc/dnsmasq.conf"
sudo brew services restart dnsmasq
```

Flush OS DNS and verify dns resolution w/ `dig`:
```
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

dig app.dolt.test

;; ANSWER SECTION:
app.dolt.test. 0 IN A 192.168.64.3
```

Restart Chrome and ensure you can again load `http://app.dolt.test`. 


#### Buidling dolt-workbench from source
Rebuilding dolt-workbench from source is quite easy should you need to make any changes. Here is a simple set of steps to get your going:
```
git clone https://github.com/dolthub/dolt-workbench.git
cd dolt-workbench

# enable Docker Buildx and build arm64 image
docker buildx create --use
docker buildx inspect --bootstrap
docker buildx build --platform linux/arm64 -t dolt-workbench:arm64 --load . --file ./docker/Dockerfile

docker tag dolt-workbench:arm64 yourusername/dolt-workbench:arm64
docker push yourusername/dolt-workbench:arm64
```

### Helpful commands

View multipass nodes:
```
multipass list
```

Viewing secrets (note: output values are base64 encoded):
```
sudo kubectl get secret dolt-credentials -n dolt-cluster-example -o yaml
```

Checking endpoints:
```
sudo kubectl get endpoints --all-namespaces
```

Checking Ingress logs for Traefik:
```
sudo kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
```

Check ingress:
```
sudo kubectl get ingress -n dolt-cluster-example
```

Shell into an container:
```
sudo kubectl exec -it -n dolt-cluster-example dolt-workbench-67656f4764-6mxbb -- sh
```

Describe a pod or view its logs for debugging purposes:
```
sudo kubectl describe pods/dolt-0 -n dolt-cluster-example
sudo kubectl logs pods/dolt-0 -n dolt-cluster-example
```

Deleting a pod:
```
sudo kubectl delete pod <pod-name> -n <namespace> 
```

Deleting our multipass environment once we are done experimenting:
```
multipass delete k3s k3s-worker
multipass purge
```

## License

Bitnode is currently licensed under the Apache 2.0 License, which explicitly grants users the right to patent derivate works, but also protects us from any form of patent retaliation. This is free and open source software. 
