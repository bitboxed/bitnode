# bitnode

Bitboxed is building better roads and bridges from web2 and traditional public cloud to a decentralized, permissionless world.

Bitnode is a core set of containers, kubernetes manifests, and utils for running the core bitnode server with various decentralized platform services: storage, compute, networking, databases, and other cloud services. These services are being developed with decentralized infrastructure and public mining & consumption in mind, and therefore will remain free and open source to the public.  

While Bitboxed may leverage some of these tools for public or hybrid cloud use cases, all private infra or otherwise closed source derivative works should remain separate. 

## Local setup with k3s

We use [k3s](https://k3s.io/) because it closely emulates k8s, its lightweight and works well at the edge, and its easy to setup on any operating system.

### Local Dev Installation

Prereqs:
Clone the repo, and you may want to [install Docker](https://www.docker.com/products/docker-desktop/) if you haven't already.
```
git clone https://github.com/bitboxed/bitnode.git
cd bitnode
```

Setup multipass and k3s. Here is a minimal set of steps to get started, or you can follow along in the guide [here](https://dev.to/chillaranand/local-kubernetes-cluster-with-k3s-on-mac-m1-i57) on setting up k3s:
```
brew install --cask multipass

# start multipass with a default dns config (helps w/ mounting)
multipass launch --name k3s --memory 2G --disk 8G --cloud-init - <<EOF
#cloud-config
write_files:
  - path: /etc/systemd/resolved.conf
    content: |
      [Resolve]
      DNS=8.8.8.8
      DNSStubListener=no
runcmd:
  - sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
  - sudo systemctl restart systemd-resolved
EOF

# mount your local bitnode directory on the VM
multipass mount ~/Github/bitnode k3s:~/bitnode

# shell in and install k3s
multipass shell k3s
ubuntu@k3s:~$ curl -sfL https://get.k3s.io | sh -

# open new terminal for k3s-worker node get token & ip of k3s for adding in the new node
multipass exec k3s sudo cat /var/lib/rancher/k3s/server/node-token
multipass info k3s | grep -i ip

# create a k3s-worker node in a new terminal and add it to the cluster (using your k3s ip and k3s token)
multipass launch --name k3s-worker --memory 2G --disk 8G --cloud-init - <<EOF
#cloud-config
write_files:
  - path: /etc/systemd/resolved.conf
    content: |
      [Resolve]
      DNS=8.8.8.8
      DNSStubListener=no
runcmd:
  - sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
  - sudo systemctl restart systemd-resolved
EOF
multipass shell k3s-worker
ubuntu@k3s-worker:~$ curl -sfL https://get.k3s.io | K3S_URL=https://<your-k3s-ip>:6443 K3S_TOKEN="hs48af...947fh4::server:3tfkwjd...4jed73" sh -

# verify the k3 nodes are running now back on the k3s ternminal
ubuntu@k3s:~$ sudo kubectl get nodes -o wide
NAME         STATUS   ROLES                  AGE   VERSION        INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k3s          Ready    control-plane,master   35h   v1.32.5+k3s1   192.168.64.2   <none>        Ubuntu 24.04.2 LTS   6.8.0-60-generic   containerd://2.0.5-k3s1.32
k3s-worker   Ready    <none>                 35h   v1.32.5+k3s1   192.168.64.3   <none>        Ubuntu 24.04.2 LTS   6.8.0-60-generic   containerd://2.0.5-k3s1.32

# go ahead & ensure your default `ubuntu` user has access to run kubectl, so that `sudo kubectl` is not needed:
ubuntu@k3s:~$ mkdir -p ~/.kube
ubuntu@k3s:~$ sudo chmod +r /etc/rancher/k3s/k3s.yaml
ubuntu@k3s:~$ cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
ubuntu@k3s:~$ sudo chown $(id -u):$(id -g) ~/.kube/config
```

### Dolt DB

[Dolt](https://github.com/dolthub/dolt) is a new SQL database that has a lot of powerful features, and enables rapid development and collaboration with "git-like" features for databases. The [storage engine](https://docs.dolthub.com/architecture/storage-engine) also makes use of a novel prolly tree structure, which enables very fast diffs (highly useful for many decentralized applications).

If your node will be running Dolt, here's how you can setup it up quickly with [Direct-to-Standby Replication](https://docs.dolthub.com/sql-reference/server/replication#replication) on your k3s nodes.

```
# shell into k3s node and create dolt cluster example namespace
multipass shell k3s

# create dolt
kubectl apply -f ~/bitnode/dolt-manifest.yaml

namespace/dolt-cluster-example created
secret/dolt-credentials created
configmap/dolt created
service/dolt-internal created
service/dolt created
service/dolt-ro created
statefulset.apps/dolt created
role.rbac.authorization.k8s.io/doltclusterctl created
serviceaccount/doltclusterctl created
rolebinding.rbac.authorization.k8s.io/doltclusterctl created

# verify dolt is running (this can take a few seconds for all pods to come online)
kubectl get pods -n dolt-cluster-example -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
dolt-0                           1/1     Running   0          18h   10.42.1.21   k3s-worker   <none>           <none>
dolt-1                           1/1     Running   0          18h   10.42.0.12   k3s          <none>           <none>
dolt-2                           1/1     Running   0          18h   10.42.1.20   k3s-worker   <none>           <none>
```

Next, we'll apply the Dolt `cluster_role=primary` label to the primary dolt pod, using [doltclusterctl](https://github.com/dolthub/doltclusterctl).
***note***: we're currently having issues w/ the arm64 version of this library, so we've compiled them and tagged them in our fork [here](https://github.com/bitboxed/doltclusterctl?tab=readme-ov-file#compilation-and-publishing-to-dockerhub).

```
kubectl run -i --tty \
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

Verify `dolthub.com/cluster_role=primary` & `dolthub.com/cluster_role=standby` labels:
```
ubuntu@k3s:~$ kubectl get pods -n dolt-cluster-example --show-labels
NAME     READY   STATUS    RESTARTS   AGE    LABELS
dolt-0   2/2     Running   0          101s   app=dolt,apps.kubernetes.io/pod-index=0,controller-revision-hash=dolt-56b5bc67b9,dolthub.com/cluster_role=primary,statefulset.kubernetes.io/pod-name=dolt-0
dolt-1   2/2     Running   0          101s   app=dolt,apps.kubernetes.io/pod-index=1,controller-revision-hash=dolt-56b5bc67b9,dolthub.com/cluster_role=standby,statefulset.kubernetes.io/pod-name=dolt-1
dolt-2   2/2     Running   0          101s   app=dolt,apps.kubernetes.io/pod-index=2,controller-revision-hash=dolt-56b5bc67b9,dolthub.com/cluster_role=standby,statefulset.kubernetes.io/pod-name=dolt-2
```

Let's quickly verify we can connect to Dolt and things are running well on the cluster.
```
ubuntu@k3s:~$ kubectl get services --all-namespaces
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
ubuntu@k3s:~$ mysql -u root -ppassword -h 10.42.1.21 -D mysql --protocol=TCP
```

If you can now login w/ the sample user (`root`) and pw (`password`) created in the secret from before, now you're up and running with Dolt on k3s! :star:

#### dolt-remotes and cluster architecture
At the time of this writing, Dolt currently does not expose remotes running in cluster / highly available mode in the same way as other types of remotes. The primary/standby remotes in the HA cluster make use a [custom private/public key bearer auth](https://docs.dolthub.com/sql-reference/server/replication#a-note-on-security) strategy locally within the primary/standby instances of the cluster, and are not accessed like other conventional dolt remotes.

Dolt's cluster mode is secure by design, and while this makes remote access nontrivial, we can follow some of the replication guidance mentioned [here](https://docs.dolthub.com/sql-reference/server/replication#direct-vs.-remote-replication) to not only improve resilience, but also improve developer accessibility. To expose dolt remotes in an easier fashion, Bitnode has opted to also automatically construct a **"remote stage"** sidecar container which lives alongside each dolt primary/standby instance, is configured as a local replica so that it receives all updates from the cluster instance it mirrors, and makes access more like that of a standard [dolt remote](https://docs.dolthub.com/sql-reference/version-control/remotes#what-are-remotes). By default, the sidecar stands up another instance of dolt sql-server which can be accessed externally using [dolt's sql-server auth](https://docs.dolthub.com/sql-reference/version-control/remotes#dolt-sql-server), however any type of dolt remote could technically be used.

This gives us a few advantages when accessing dolt in a highly available cluster:
* our cluster instances of dolt can remain focused on serving read/write traffic, and handling failover in a graceful way should it occur
* our **remote stage** sidecar remote gives developers an easy way to access and clone our latest dolt data externally, make our changes in a branch downstream of the cluster, and then push our changes up to the cluster's primary `main` branch when ready (and those changes in turn will also then be applied automatically to our standby instances and our **remote stage**'s replica). In a sense, this is quite similar to Github's classic "fork and pull" workflow.
* we minimize any noise or disruption on the cluster instances serving traffic, and can instead review and merge our changes outside on the **remote stage**, and promote them to the cluster when ready.
* we have another remote copy, just in case of catastrophic failure on the cluster instances. This remote also can more easily be used for constructing [dolt backups](https://docs.dolthub.com/sql-reference/server/backups#dolt-backup-command) (both remote based and file-based backups).

```mermaid
flowchart TD
    %% External Developer
    Dev[Developer User]
    Traefik[Traefik IngressRouteTCP]

    %% Developer connects through Traefik
    Dev -->|"TCP to dolt-remote"| Traefik

    %% Traefik forwards to dolt-remote containers
    Traefik -->|"Clone access"| DR0
    Traefik -->|"Clone access"| DR1
    Traefik -->|"Clone access"| DR2

    %% Internal Cluster
    subgraph K8sCluster["Kubernetes Cluster"]
        
        subgraph InternalServices["Internal Services"]
            ServiceA[serviceA]
            ServiceB[serviceB]
        end

        subgraph ProxySQL_Pod["ProxySQL Pod"]
            Proxy[ProxySQL]
        end

        subgraph Dolt0["Dolt-0 | Primary"]
            D0[Dolt Container]
            DR0[Dolt-Remote Container]
        end

        subgraph Dolt1["Dolt-1 | Standby"]
            D1[Dolt Container]
            DR1[Dolt-Remote Container]
        end

        subgraph Dolt2["Dolt-2 | Standby"]
            D2[Dolt Container]
            DR2[Dolt-Remote Container]
        end

        %% Internal Services use ProxySQL
        ServiceA -->|"Reads/Writes"| Proxy
        ServiceB -->|"Reads/Writes"| Proxy

        %% ProxySQL traffic routing
        Proxy -->|"Write Traffic"| D0
        Proxy -->|"Read Traffic"| D1
        Proxy -->|"Read Traffic"| D2

        %% Internal Replication
        D0 -->|"Replicates to"| D1
        D0 -->|"Replicates to"| D2

        %% Dolt to Remote replication
        D0 -->|"Mirrors to"| DR0
        D1 -->|"Mirrors to"| DR1
        D2 -->|"Mirrors to"| DR2
    end
```


Here is a deeper breakdown of the remote branch "fork and pull" workflow with dolt in this architecture:
```mermaid
flowchart TD
    %% External developer
    Dev[Developer User]

    %% Cluster setup
    subgraph Cluster["Dolt Cluster"]
        subgraph DoltPrimary["Dolt Primary"]
            D0[Dolt Container - Primary]
            D0HTTP[TCP/HTTP Remote<br>dolt-0.dolt-internal:50051/upstream_db]
        end

        subgraph DoltRemote["Dolt-Remote Container"]
            FR[File Remote<br>/dbremote/upstream_db]
            HTTPRemote[TCP/HTTP Remote<br>primary.dolt.test:8090/downstream_db]
        end

        subgraph Standbys["Dolt Standby Nodes"]
            D1[Dolt-1 Standby]
            D1HTTP[TCP/HTTP Remote<br>dolt-1.dolt-internal:50051/upstream_db]
            D2[Dolt-2 Standby]
            D2HTTP[TCP/HTTP Remote<br>dolt-2.dolt-internal:50051/upstream_db]
        end
    end

    %% Step 1: DB created, replicated to file:// remote
    D0 -- "New 'upstream_db' replicated" --> FR
    D0 -- "All changes replicated to file remote" --> FR

    %% Step 2: Clone downstream TCP/HTTP Remote
    FR -->|"Cloned into downstream_db"| HTTPRemote
    HTTPRemote -->|"Clone downstream_db"| Dev

    %% Step 3: Developer feature work
    Dev -->|"Push feature-branch"| HTTPRemote
    HTTPRemote -->|"Push feature-branch to shared file remote"| FR

    %% Step 4: Pull + merge into upstream
    FR -->|"Fetch feature-branch from file remote"| D0HTTP
    D0HTTP -->|"Merge feature-branch to 'main'"| D0

    %% Step 5: Replication to standbys
    D0 -->|"Replicates 'main' to"| D1
    D0 -->|"Replicates 'main' to"| D2

    %% Step 6: Mirror merged main back to file remote
    D0 -->|"Branch 'main' updates replicate to file remote"| FR

    %% Step 7: Pull main into downstream_db
    FR -->|"Pull main into downstream_db"| HTTPRemote

    %% Step 8: Pull main into downstream_db
    HTTPRemote -->|"Pull main"| Dev

    %% Internal TCP/HTTP Remotes
    D1 --> D1HTTP
    D2 --> D2HTTP
```

#### dolt-remotes setup & clone access

To enable dolt-remote access to your cluster, you can now do the following:

First, we'll expose a new TCP port with Traefik by [extending the Traefik Ingress controller](https://docs.k3s.io/helm?_highlight=traefik&_highlight=config.yaml#customizing-packaged-components-with-helmchartconfig) via HelmChartConfig:
```
# create a new manifest extension file here:
sudo mkdir -p /var/lib/rancher/k3s/server/manifests
sudo vi /var/lib/rancher/k3s/server/manifests/traefik-config.yaml

# add the following manifest 
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      dolt:
        port: 8090
        expose:
          default: true
        protocol: TCP
    additionalArguments:
      - --entrypoints.dolt.address=:8090/tcp
```

Traefik should restart automatically, but we can also restart it manually. Review logs and ensure it comes up healthy with port 8090 exposed:
```
ubuntu@k3s:~$ kubectl -n kube-system rollout restart deploy/traefik
deployment.apps/traefik restarted

ubuntu@k3s:~$ kubectl get pods -n kube-system
NAME                                      READY   STATUS      RESTARTS   AGE
coredns-697968c856-crpnr                  1/1     Running     0          24h
helm-install-traefik-bkf4m                0/1     Completed   0          54s
helm-install-traefik-crd-f5xzk            0/1     Completed   0          24h
local-path-provisioner-774c6665dc-dbq2q   1/1     Running     0          24h
metrics-server-6f4c6675d5-28wcx           1/1     Running     0          24h
svclb-traefik-be138b09-272v9              3/3     Running     0          48s
svclb-traefik-be138b09-rp67m              3/3     Running     0          50s
traefik-68d6759d54-5zgx9                  1/1     Running     0          11s

ubuntu@k3s:~$ kubectl logs traefik-6cb94965c7-4v67n -n kube-system
2025-06-25T15:03:59Z INF Traefik version 3.3.6 built on 2025-04-18T09:18:46Z version=3.3.6
2025-06-25T15:03:59Z INF Stats collection is enabled.
2025-06-25T15:03:59Z INF Starting provider aggregator *aggregator.ProviderAggregator
2025-06-25T15:03:59Z INF Starting provider *traefik.Provider
2025-06-25T15:03:59Z INF Starting provider *ingress.Provider
2025-06-25T15:03:59Z INF ingress label selector is: "" providerName=kubernetes
2025-06-25T15:03:59Z INF Creating in-cluster Provider client providerName=kubernetes
2025-06-25T15:03:59Z INF Starting provider *crd.Provider
2025-06-25T15:03:59Z INF label selector is: "" providerName=kubernetescrd
2025-06-25T15:03:59Z INF Creating in-cluster Provider client providerName=kubernetescrd
2025-06-25T15:03:59Z INF Starting provider *acme.ChallengeTLSALPN

ubuntu@k3s:~$ kubectl -n kube-system describe svc traefik | grep 8090
Port:                     dolt  8090/TCP
Endpoints:                10.42.0.13:8090
```

Apply `~/bitnode/dolt-remotes.yaml` to expose the primary dolt remote stage as a service and as an ingress tcp route.
**important**: ensure you have already assigned a primary label to one of the dolt pods using the doltclusterctl `applyprimarylabels` command mentioned above prior.
```
ubuntu@k3s:~$ kubectl apply -f ~/bitnode/dolt-remotes.yaml
service/dolt-primary-remotesapi created
ingressroutetcp.traefik.io/dolt-primary-remotesapi-tcp created
```

Let's next update `/etc/hosts` so that we can target our remote externally:
```
$ multipass info k3s-worker | grep IPv4 | awk '{print $2}'
192.168.64.22

# add the second line targeting your k3s-worker ip:
$ sudo vi /etc/hosts 
 127.0.0.1   localhost
 192.168.64.22 primary.dolt.test standby.dolt.test

# flush dns
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

Validate we can now reach the remote (we expect a HTTP 400 when auth or a db is not yet specified):
```
$ curl -v http://primary.dolt.test:8090
> GET / HTTP/1.1
> Host: primary.dolt.test:8090
> User-Agent: curl/8.7.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 400 Bad Request
```

With our remote now in place on the primary, let's create a dolt database on the primary, clone the database downstream, grant access to a new sql user, and test a simple git workflow. You can refer dolt's [replicate guide](https://docs.dolthub.com/sql-reference/server/replication#replication-through-a-remote), [sql-server remote guide](https://docs.dolthub.com/sql-reference/version-control/remotes#reading-from-sql-server), and the [How to Use Dolt SQL Remotes blog](https://www.dolthub.com/blog/2021-09-22-sql-remotes/) during this process for more details.

create database on the primary (`dolt-0` in this case):
```
ubuntu@k3s:~$ kubectl exec -it -n dolt-cluster-example dolt-0 -- bash
root@dolt-0:/db# dolt sql -q "create database upstream_db;"

root@dolt-0:/db# dolt sql -q "select * from dolt_remotes"
+------------+-----------------------------------------------+--------------------------------------------+--------+
| name       | url                                           | fetch_specs                                | params |
+------------+-----------------------------------------------+--------------------------------------------+--------+
| dolt-1     | http://dolt-1.dolt-internal:50051/upstream_db | ["refs/heads/*:refs/remotes/dolt-1/*"]     | null   |
| dolt-2     | http://dolt-2.dolt-internal:50051/upstream_db | ["refs/heads/*:refs/remotes/dolt-2/*"]     | null   |
| downstream | file:///dbremote/upstream_db                  | ["refs/heads/*:refs/remotes/downstream/*"] | null   |
+------------+-----------------------------------------------+--------------------------------------------+--------+

# create a new table for demo purposes
root@dolt-0:/db# dolt sql -q "create table table_a (a int primary key)"
root@dolt-0:/db# dolt add -A
root@dolt-0:/db# dolt commit -am "add table_a"
  commit f8tnhmcdpfc8jci47ksmv3rhtmbk944d (HEAD -> main)
  Author: dolt kubernetes deployment <dolt@kubernetes.deployment>
  Date:  Wed Jun 25 20:04:36 +0000 2025
    add table_a
```

This will auto replicate to our dolt-remote due to the following settings in [dolt-manifest.yaml](./dolt-manifest.yaml).
```
dolt sql -q "set @@persist.dolt_replicate_to_remote = 'downstream'"
dolt sql -q "set @@persist.dolt_replication_remote_url_template = 'file:///dbremote/{database}';"
```

verify it was replicated on the dolt-remote container:
```
# shell into dolt-remote container
ubuntu@k3s:~$ kubectl exec -it -n dolt-cluster-example dolt-0 -c dolt-remote -- bash
root@dolt-0:/dbremote# ls
config.yaml  upstream_db
```

create our downstream clone of the upstream:
```
root@dolt-0:/dbremote# dolt clone file://./upstream_db downstream_db
cloning file:///dbremote/upstream_db
11 of 11 chunks complete.
```

ensure our downstream auto replicates `main` branch "on read" from the upstream. We'll also use the same dolt commit user name as the upstream for consistency.
```
root@dolt-0:/dbremote cd downstream_db
root@dolt-0:/dbremote/downstream_db# dolt sql -q "set @@persist.dolt_replicate_heads = 'main'"
root@dolt-0:/dbremote/downstream_db# dolt sql -q "set @@persist.dolt_read_replica_remote = 'origin'"
root@dolt-0:/dbremote/downstream_db# dolt config --global --add user.name "dolt kubernetes deployment"
root@dolt-0:/dbremote/downstream_db# dolt config --global --add user.email "dolt@kubernetes.deployment"
```

create a new sql user with clone & write permissions: 
```
# create sql test user with clone & write permissions
root@dolt-0:/dbremote/downstream_db# dolt sql -q "create user 'exampleuser'@'%' identified by 'examplepassword'"
root@dolt-0:/dbremote/downstream_db# dolt sql -q "grant clone_admin on *.* to 'exampleuser'@'%'"
root@dolt-0:/dbremote/downstream_db# dolt sql -q "GRANT ALL PRIVILEGES ON *.* TO 'exampleuser'@'%' WITH GRANT OPTION"
```
note: sql users and grants are not replicated and remain unique to the database (see [dolt replication behavior](https://docs.dolthub.com/sql-reference/server/replication#replication-behavior) mention).

With the downstream_db in place, let's restart dolt sql-server process so that it recognizes the new database moving forward:
```
root@dolt-0:/dbremote/downstream_db# exit
exit
ubuntu@k3s:~$ kubectl exec dolt-0 -n dolt-cluster-example -c dolt-remote -- /bin/sh -c 'pkill dolt && /usr/local/bin/dolt sql-server --config config.yaml &'
```

Now externally from the cluster, let's clone our downstream_db with sql auth and make a new feature.
```
$ DOLT_REMOTE_PASSWORD=examplepassword dolt clone --user exampleuser http://primary.dolt.test:8090/downstream_db downstream_db
cloning http://primary.dolt.test:8090/downstream_db
20 of 20 chunks complete.

$ cd downstream_db
# validate we have the latest upstream_db changes
$ dolt log
commit f8tnhmcdpfc8jci47ksmv3rhtmbk944d (HEAD -> main, remotes/origin/main)
Author: dolt kubernetes deployment <dolt@kubernetes.deployment>
Date:  Wed Jun 25 16:04:36 -0400 2025
        add table_a

commit c5cgsbjnvmiq8bcvtj16lrq418fue9os
Author: dolt kubernetes deployment <dolt@kubernetes.deployment>
Date:  Wed Jun 25 15:49:35 -0400 2025
        Initialize data repository

$ dolt checkout -b feature-branch
Switched to branch 'feature-branch'

$ dolt sql -q "create table table_b (b int primary key)"
$ dolt add -A
$ dolt commit -m "add table_b";
```

Push our feature branch up to the dolt-remote:
```
$ DOLT_REMOTE_PASSWORD=examplepassword dolt push origin --user exampleuser feature-branch:feature-branch
\ Uploading...To http://primary.dolt.test:8090/downstream_db
 * [new branch]          feature-branch -> feature-branch
```

With our featre branch pushed to dolt-remote, let's go back to our dolt-remote container and now push it to the upstream_db:
```
root@dolt-0:/dbremote/downstream_db# dolt sql -q "select * from dolt_remotes"
+--------+------------------------------+----------------------------------------+--------+
| name   | url                          | fetch_specs                            | params |
+--------+------------------------------+----------------------------------------+--------+
| origin | file:///dbremote/upstream_db | ["refs/heads/*:refs/remotes/origin/*"] | {}     |
+--------+------------------------------+----------------------------------------+--------+

root@dolt-0:/dbremote/downstream_db# dolt push origin feature-branch
- Uploading...To file:///dbremote/upstream_db
 * [new branch]          feature-branch -> feature-branch
 ```

 Great! :ok_hand: ok, let's fetch our feature branch onto the primary now and merge it to `main`:
 ```
 # back on the primary on dolt-0
 root@dolt-0:/db/upstream_db# dolt fetch downstream feature-branch
 root@dolt-0:/db/upstream_db# dolt sql
 upstream_db/main> CALL dolt_merge('downstream/feature-branch');
+----------------------------------+--------------+-----------+------------------+
| hash                             | fast_forward | conflicts | message          |
+----------------------------------+--------------+-----------+------------------+
| 440r0pfa2pnqe96d2mclg4slcs74our0 | 1            | 0         | merge successful |
+----------------------------------+--------------+-----------+------------------+
1 row in set (0.05 sec)
 ```

Fantastic! Our first dolt merge is now completed. :balloon: 

You can read much more about dolt [merges](https://docs.dolthub.com/sql-reference/version-control/merges) here if you're now curious.

Ok, this is all working really well, but what about when changes have been made upstream by real user traffic, and we want to pull those changes back freshly into our downstream clone?

Let's do just that, and simulate a change first on the primary.
```
upstream_db/main> insert into table_a (a) values (10);
Empty set (0.03 sec)

upstream_db/main*> insert into table_b (b) values (20);
Empty set (0.02 sec)
```

Given we did not set the [persist.dolt_transaction_commit](https://docs.dolthub.com/sql-reference/server/replication#configuring-a-primary) to `1`, we'll need to make an additional dolt commit for the downstream to pick up our latest changes. Otherwise, it would happen on the next read downstream automatically.
```
upstream_db/main*> exit
Bye
root@dolt-0:/db/upstream_db# dolt add -A
root@dolt-0:/db/upstream_db# dolt commit -m "added data upstream";
  commit e9aftmoifmbbe8punc188q0oedn1c01i (HEAD -> main)
  Author: dolt kubernetes deployment <dolt@kubernetes.deployment>
  Date:  Wed Jun 25 21:01:35 +0000 2025

    added data upstream
```

Ok, let's switch back over to our downstream_db on the dolt-remote now and pull the latest changes in `main` so that our downstream developers can consume them easily.
```
root@dolt-0:/dbremote/downstream_db# dolt pull origin main
\ Pulling...Updating e9aftmoifmbbe8punc188q0oedn1c01i..e9aftmoifmbbe8punc188q0oedn1c01i
\ Pulling...Everything up-to-date
```

Beautiful! Let's go back to our external clone now outside of the cluster, and validate we can pull the latest changes and see them.
```
$  DOLT_REMOTE_PASSWORD=examplepassword dolt pull origin main --user exampleuser
\ Pulling...Fast-forward
Updating 440r0pfa2pnqe96d2mclg4slcs74our0..e9aftmoifmbbe8punc188q0oedn1c01i
Everything up-to-date

$ dolt log -n 1
commit e9aftmoifmbbe8punc188q0oedn1c01i (HEAD -> feature-branch, remotes/origin/main)
Author: dolt kubernetes deployment <dolt@kubernetes.deployment>
Date:  Wed Jun 25 17:01:35 -0400 2025
        added data upstream

$ dolt sql -q "select * from table_a"
+----+
| a  |
+----+
| 10 |
+----+

$ dolt sql -q "select * from table_b"
+----+
| b  |
+----+
| 20 |
+----+
```

Wonderful!! :tada: 

You've now completed a basic git developer flow with dolt and dolt-remotes inside a highly available cluster. :star2:

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
ubuntu@k3s:~$ kubectl apply -f ~/bitnode/dolt-workbench.yaml

deployment.apps/dolt-workbench created
service/dolt-workbench created
ingress.networking.k8s.io/dolt-workbench-app-ingress created
ingress.networking.k8s.io/dolt-workbench-api-ingress created
```

Afterwards, you should now be able to view Dolt Workbench at `http://app.dolt.test/` and login to your dolt databases locally within the cluster using the admin user/pw secret mentioned before, e.g: `mysql://root:password@dolt:3306` (***note***: `dolt` is service name for our primary dolt database and can be used as the host local to the cluster).
<img width="1233" alt="Screenshot 2025-06-05 at 9 47 18 PM" src="https://github.com/user-attachments/assets/90e5b174-feca-4420-89eb-aacd36138f45" />
<img width="1233" alt="Screenshot 2025-06-05 at 9 47 40 PM" src="https://github.com/user-attachments/assets/4ea840f1-4421-437e-a3d6-0c50aca23ff3" />
<img width="1233" alt="Screenshot 2025-06-05 at 9 47 46 PM" src="https://github.com/user-attachments/assets/c4689aa4-277d-4dec-9785-718351886584" />
<img width="1233" alt="Screenshot 2025-06-05 at 9 48 02 PM" src="https://github.com/user-attachments/assets/54088319-8f3d-4fbe-a20d-d89fdac0981a" />
<img width="1233" alt="Screenshot 2025-06-05 at 9 48 09 PM" src="https://github.com/user-attachments/assets/250f063c-64a4-44e3-a213-d7c7d415008e" />



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

#### ProxySQL setup with Dolt DB

The best way to network with your active/passive Dolt cluster is now via a proxy which can proxy read and write traffic.

Here's how to quickly get proxysql running with sample dolt cluster here:
```
ubuntu@k3s:~$ kubectl apply -f ~/bitnode/dolt-proxysql.yaml
configmap/proxysql-config created
deployment.apps/proxysql created
service/proxysql created
```

This will do the following:
* create a proxysql service pod on port `3306` and configure its `mysql_servers` with dolt and dolt-ro db's (active/passive)
* create a `monitor` user within dolt that proxysql can connect to for monitoring purposes
* create a sample `stnduser` that we can use to connect to the ProxySQL server with to perform queries (note: this user is also created in dolt in addition to the ProxySQL `mysql_users` table)

Let's validate this now:

```
ubuntu@k3s:~$ kubectl get pods -n dolt-cluster-example -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP           NODE         NOMINATED NODE   READINESS GATES
dolt-0                            1/1     Running   0          3d3h   10.42.1.4    k3s-worker   <none>           <none>
dolt-1                            1/1     Running   0          3d3h   10.42.0.10   k3s          <none>           <none>
dolt-2                            1/1     Running   0          3d3h   10.42.1.6    k3s-worker   <none>           <none>
dolt-workbench-65d7b5b845-qn6kn   1/1     Running   0          31m    10.42.1.18   k3s-worker   <none>           <none>
proxysql-788f9987b5-xm6n2         1/1     Running   0          6s     10.42.1.23   k3s-worker   <none>           <none>

# validate proxysql `monitor` user created in dolt-0 (the same can be done in dolt-1, dolt-2)
ubuntu@k3s:~$ mysql -u root -ppassword -h 10.42.1.4 -D mysql
mysql> SELECT user, host, plugin FROM mysql.user;
+---------------------+-----------+-----------------------+
| user                | host      | plugin                |
+---------------------+-----------+-----------------------+
| stnduser            | %         | mysql_native_password |
| root                | %         | mysql_native_password |
| event_scheduler     | localhost | mysql_native_password |
| __dolt_local_user__ | localhost | mysql_native_password |
| monitor             | %         | mysql_native_password |
+---------------------+-----------+-----------------------+


# connect to proxysql mysql instance and review mysql_servers
ubuntu@k3s:~ kubectl exec -it -n dolt-cluster-example <proxysql-pod> -- mysql -h 127.0.0.1 -P6032 -uadmin -padmin

MySQL [(none)]> select * from mysql_users;
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+------------+---------+
| username | password | active | use_ssl | default_hostgroup | default_schema | schema_locked | transaction_persistent | fast_forward | backend | frontend | max_connections | attributes | comment |
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+------------+---------+
| monitor  | monitor  | 1      | 0       | 10                |                | 0             | 1                      | 0            | 1       | 1        | 10000           |            |         |
| stnduser | stnduser | 1      | 0       | 10                |                | 0             | 1                      | 0            | 1       | 1        | 10000           |            |         |
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+------------+---------+

MySQL [(none)]> SELECT * FROM mysql_servers;
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 10           | dolt     | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 20           | dolt-ro  | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+

MySQL [(none)]> SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 3;
+----------+------+------------------+-------------------------+---------------+
| hostname | port | time_start_us    | connect_success_time_us | connect_error |
+----------+------+------------------+-------------------------+---------------+
| dolt-ro  | 3306 | 1749446009975247 | 2888                    | NULL          |
| dolt     | 3306 | 1749446009963602 | 1420                    | NULL          |
| dolt-ro  | 3306 | 1749445889975643 | 2304                    | NULL          |
+----------+------+------------------+-------------------------+---------------+

MySQL [(none)]> SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 3;
+----------+------+------------------+----------------------+------------+
| hostname | port | time_start_us    | ping_success_time_us | ping_error |
+----------+------+------------------+----------------------+------------+
| dolt     | 3306 | 1749446123406597 | 556                  | NULL       |
| dolt-ro  | 3306 | 1749446123405973 | 1352                 | NULL       |
| dolt     | 3306 | 1749446115405269 | 334                  | NULL       |
+----------+------+------------------+----------------------+------------+
```

With your cluster configured correctly for ProxySQL, we can now test a simple write to ProxySQL using our `stnduser`:

```
# connect to proxysql as stnduser
ubuntu@k3s:~$ kubectl exec -it -n dolt-cluster-example proxysql-cb8c95595-q9z9n -- mysql -h 127.0.0.1 -P6033 -ustnduser -pstnduser

MySQL [(none)]> CREATE DATABASE dolt_sample_db;
Query OK, 1 row affected (0.101 sec)

MySQL [(none)]> USE dolt_sample_db;
Database changed

MySQL [dolt_sample_db]> create table t1 (pk int primary key);
Query OK, 0 rows affected (0.008 sec)
```

Validate the write was replicated:
```
ubuntu@k3s:~$ mysql -u stnduser -pstnduser -h <dolt-0-ip-address> -D mysql
mysql> use dolt_sample_db;
Database changed
mysql> show tables;
+--------------------------+
| Tables_in_dolt_sample_db |
+--------------------------+
| t1                       |
+--------------------------+
1 row in set (0.01 sec)

# add a dolt commit
mysql> call dolt_commit('-Am', 'create table t1');
+----------------------------------+
| hash                             |
+----------------------------------+
| kebsuag1fo5ueplgfd2tea9cke65fp6v |
+----------------------------------+
1 row in set (0.01 sec)

# validate on the read-only dolt-1 db that the commit was added
ubuntu@k3s:~$ mysql -u stnduser -pstnduser -h <dolt-1-ip-address> -D mysql
mysql> use dolt_sample_db;
Database changed
mysql> show tables;
+--------------------------+
| Tables_in_dolt_sample_db |
+--------------------------+
| t1                       |
+--------------------------+
1 row in set (0.01 sec)

# validate its read-only
mysql> call dolt_commit('-Am', 'create table t1');
ERROR 1105 (HY000): database server is set to read only mode

mysql> select * from dolt_log;
+----------------------------------+----------------------------+----------------------------+---------------------+----------------------------+--------------+
| commit_hash                      | committer                  | email                      | date                | message                    | commit_order |
+----------------------------------+----------------------------+----------------------------+---------------------+----------------------------+--------------+
| kebsuag1fo5ueplgfd2tea9cke65fp6v | stnduser                   | stnduser@%                 | 2025-06-09 22:57:58 | create table t1            |            2 |
| is20hjt3l004n94lb81267qusnu707ei | dolt kubernetes deployment | dolt@kubernetes.deployment | 2025-06-09 22:51:10 | Initialize data repository |            1 |
+----------------------------------+----------------------------+----------------------------+---------------------+----------------------------+--------------+
```

#### Dolt failover

At the time of this writing, a formal kubernetes operator does not yet exist, but a simple Dolt failover cron job has been provided, which will attempt to fail the primary dolt db over to the standby db in case of primary failure. This job currently runs the `promotestandby` command, which is more aggressive than the `gracefulfailover` command (and will still succeed if a new standby is not assigned or is down). You can read more about doltclusterctl operations [here](https://github.com/bitboxed/doltclusterctl?tab=readme-ov-file#operations).

Usage:
```
ubuntu@k3s:~$ kubectl apply -f ~/bitnode/dolt-failover.yaml
cronjob.batch/dolt-failover-job created
cronjob.batch/dolt-failover-monitor created

ubuntu@k3s:~$ kubectl get cronjobs -n dolt-cluster-example
NAME                    SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
dolt-failover-job       @yearly       <none>     True      0        <none>          2m23s
dolt-failover-monitor   */1 * * * *   <none>     False     0        45s             68m

ubuntu@k3s:~$ kubectl get jobs -n dolt-cluster-example
dolt-failover-monitor-29162827   Complete   1/1           3s         66s
dolt-failover-monitor-29162828   Complete   1/1           3s         6s

ubuntu@k3s:~$ kubectl logs jobs/dolt-failover-monitor-29162828 -n dolt-cluster-example
Dolt primary is healthy.
```

We should see the "Dolt primary is healthy" message when the cluster is healthy. By default, the monitor cron job runs every minute.

We can manually fail the primary over to test by doing the following:
```
ubuntu@k3s:~$ kubectl create job --from=cronjob/dolt-failover-job dolt-failover-manual -n dolt-cluster-example
job.batch/dolt-failover-manual created

ubuntu@k3s:~$ kubectl logs jobs/dolt-failover-manual -n dolt-cluster-example
[*] Running doltclusterctl promotestandby
2025/06/12 23:26:26 running promotestandby against dolt-cluster-example/dolt
2025/06/12 23:26:26 found standby to promote: dolt-cluster-example/dolt-1
2025/06/12 23:26:26 labeled all pods as standby
2025/06/12 23:26:26 called dolt_assume_cluster_role primary on dolt-cluster-example/dolt-1
2025/06/12 23:26:26 applied primary label to dolt-cluster-example/dolt-1
[✓] Failover complete.

# reboot the proxysql pod (this is already handled in the monitor cron job)
ubuntu@k3s:~$ kubectl rollout restart deployment proxysql -n dolt-cluster-example
deployment.apps/proxysql restarted
```

Now verify the failover worked as expected:
```
# verify dolt-1 now has the cluster_role=primary label
ubuntu@k3s:~$ kubectl get pods  -n dolt-cluster-example --show-labels
NAME                                   READY   STATUS      RESTARTS   AGE     LABELS
dolt-0                                 1/1     Running     0          3d      app=dolt,apps.kubernetes.io/pod-index=0,controller-revision-hash=dolt-747bfdfc7d,dolthub.com/cluster_role=standby,statefulset.kubernetes.io/pod-name=dolt-0
dolt-1                                 1/1     Running     0          3d      app=dolt,apps.kubernetes.io/pod-index=1,controller-revision-hash=dolt-747bfdfc7d,dolthub.com/cluster_role=primary,statefulset.kubernetes.io/pod-name=dolt-1
dolt-2                                 1/1     Running     0          3d      app=dolt,apps.kubernetes.io/pod-index=2,controller-revision-hash=dolt-747bfdfc7d,dolthub.com/cluster_role=standby,statefulset.kubernetes.io/pod-name=dolt-2

# verify we can still write with proxysql
# note: the dolt/dolt-ro service names and service cluster_role labels used in proxysql automatically ensure the correct dbs are targeted after failover

ubuntu@k3s:~$ kubectl exec -it -n dolt-cluster-example <proxysql-pod-name> -- mysql -h 127.0.0.1 -P6033 -ustnduser -pstnduser
MySQL [(none)]> use dolt_sample_db;
Database changed
MySQL [dolt_sample_db]> create table t2 (pk int primary key);
Query OK, 0 rows affected (0.020 sec)

# validate the write occurred and still replicated in the dolt db's...
ubuntu@k3s:~$ mysql -u stnduser -pstnduser -h <dolt-db-ip> -D mysql
MySQL [(none)]> use dolt_sample_db;
Database changed
mysql> show tables;
+--------------------------+
| Tables_in_dolt_sample_db |
+--------------------------+
| t1                       |
| t2                       |
+--------------------------+
```


### Helpful commands

View multipass nodes:
```
multipass list
```

Viewing secrets (note: output values are base64 encoded):
```
kubectl get secret dolt-credentials -n dolt-cluster-example -o yaml
```

Checking endpoints:
```
kubectl get endpoints --all-namespaces
```

Checking Ingress logs for Traefik:
```
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
```

Check ingress:
```
kubectl get ingress -n dolt-cluster-example
```

Shell into an container:
```
kubectl exec -it -n dolt-cluster-example dolt-workbench-67656f4764-6mxbb -- sh
```

Shell into a db container:
```
kubectl exec -it -n dolt-cluster-example dolt-0 -- bash
```

Shell into the remote db container:
```
kubectl exec -it -n dolt-cluster-example dolt-0 -c dolt-remote -- bash
```

Run a debug container with bash:
```
kubectl -n dolt-cluster-example run -it --rm debug --image=ubuntu -- bash

# or sh..
kubectl -n dolt-cluster-example run -it --rm debug --image=alpine -- sh
```

Describe a pod or view its logs for debugging purposes:
```
kubectl describe pods/dolt-0 -n dolt-cluster-example
kubectl logs pods/dolt-0 -n dolt-cluster-example
```

Deleting your entire namespace so you can "start over" in k3s:
```
kubectl delete all -n dolt-cluster-example --all
kubectl apply -f ~/bitnode/dolt-manifest.yaml
```

Deleting a pod:
```
kubectl delete pod <pod-name> -n <namespace> 
```

Deleting our multipass environment once we are done experimenting:
```
multipass delete k3s k3s-worker
multipass purge
```

## License

Bitnode is currently licensed under the Apache 2.0 License, which explicitly grants users the right to patent their derivate works, but also protects us from any form of patent retaliation should we choose to patent our own derivatives. This is free and open source software. 
