# Three node desktop RKE2 cluster

This scenario creates a three-node desktop RKE2 cluster consisting of three VMs: one control plane node, and two workers. This is a disconnected install: All that is required to install RKE2 are two tars, one image list file, and the RKE2 install script - all downloaded from the RKE2 release site. The scripting is done in Bash. The RKE2 air-gapped install instructions are at:

https://docs.rke2.io/install/airgap/

## Pre-requisites

1. Three VMs installed with CentOS Stream (it's possible other CentOS versions would work but I've only tested with CentOS Stream)
2. An ssh private key matching the corresponding public key in each VM
3. The RKE2 binaries and install script exist in the binaries directory of this repo

To populate the binaries directory, per the RKE2 docs, use the following curl commands with your current working directory the root of this reop. E.g. `~/projects/rke2`.

```
$ export RKE2VER=v1.21.5

$ curl -sfL https://github.com/rancher/rke2/releases/download/$RKE2VER%2Brke2r2/rke2-images.linux-amd64.tar.zst\
  -o ./binaries/rke2-images.linux-amd64.tar.zst &&\
  curl -sfL https://github.com/rancher/rke2/releases/download/$RKE2VER%2Brke2r2/rke2.linux-amd64.tar.gz\
  -o ./binaries/rke2.linux-amd64.tar.gz &&\
  curl -sfL https://github.com/rancher/rke2/releases/download/$RKE2VER%2Brke2r2/sha256sum-amd64.txt\
  -o ./binaries/sha256sum-amd64.txt &&\
  curl -sfL https://get.rke2.io -o ./binaries/install.sh
```

_Note: %2B in the URLs above is an encoded plus sign (+)._

## Create the VMs

This project uses scripts developed as part of the https://github.com/aceeric/desktop-kubernetes project. That project stands up a three-VM k8s cluster using VirtualBox, the CentOS Stream ISO, and the various upstream Kubernetes components. The _Desktop Kubernetes_ project also has functionality that simplifies creating VirtualBox VMs. So this project uses that functionality.

You can create the VMs however you want. The way I created then with _Desktop Kubernetes_ is shown for documentation purposes.

### Use Desktop Kubernetes to create the VMs

This section assumes you've git cloned _Desktop Kubernetes_ into `~/projects/desktop-kubernetes` and that is your current working directory. If you git cloned differently, adjust any instructions.

First create a template VM. This VM will have CentOS Stream and Virtual Box Guest Additions. This example uses VirtualBox bridge networking via the `--host-network-interface` arg. I had trouble with the host-only networking in VBox and am still researching a resolution so for now, the VMs must be configured with bridge networking or the RKE2 agent doesn't start. The value for the host network interface is unique to each workstation. Use `ifconfig` to get your primary network interface name.

```
hack/scripts/create-template-vm --host-network-interface enp0s31f6 rke2-template /sdb1/virtualboxvms
```

The above statement creates a template VM named `rke2-template`. Next create three clones from the template:

```
for clone in rke1 rke2 rke3; do \
  hack/scripts/create-clone-vm --host-network-interface enp0s31f6 rke2-template $clone /sdb1/virtualboxvms; \
done
```

You now have three Virtual Box VMs running named `rke1`, `rke2`, and `rke3`. Get the IP addresses for the VMs:

```
$ for host in rke1 rke2 rke3; do echo export $host=$(xec get-vm-ip $host); done
export rke1=192.168.0.31
export rke2=192.168.0.32
export rke3=192.168.0.33
```

Finally: when you generate a VM using _Desktop Kubernetes_ it creates an SSH key pair and places the public key into each VM and the private key in `~/projects/desktop-kubernetes/generated/kickstart/id_ed25519`. You need that SSH key to access the VMs here. So:

```
export PRIVKEY=~/projects/desktop-kubernetes/generated/kickstart/id_ed25519
```

### In this (rke2) Project

From here forward, the instructions assume you've git cloned https://github.com/aceeric/rke2 into `~/projects/rke2` and that is your current working directory.

Set environment variables for the VMs using the values above

```
export rke1=192.168.0.31
export rke2=192.168.0.32
export rke3=192.168.0.33
```

## Install RKE2 server

### Copy the upstream binaries and this project's install script to the VM

```
ssh -i $PRIVKEY root@$rke1 "mkdir /root/rke2-artifacts"
scp -i $PRIVKEY binaries/* scripts/install-rke2* root@$rke1:/root/rke2-artifacts
```

### Run the RKE2 installer to install the server

```
ssh -i $PRIVKEY root@$rke1 "/root/rke2-artifacts/install-rke2-server"
```

If success:

```
Finished. To access the RKE2 cluster from outside the VM:
scp -i $PRIVKEY root@192.168.56.120:/etc/rancher/rke2/rke2.yaml ~/.kube/rke2-config
sed -i 's/127.0.0.1/192.168.56.120/g' ~/.kube/rke2-config
export KUBECONFIG=~/.kube/rke2-config
```

On my system, which has the following CPU config - Intel i7-8700 CPU @ 3.20GHz x 12   - generating the RKE2 server in the VM takes about four minutes.

Execute the following commands to configure your workstation to access the cluster VMs - which at this point consists of a single control plane node:

```
scp -i $PRIVKEY root@$rke1:/etc/rancher/rke2/rke2.yaml ~/.kube/rke2-config
sed -i "s/127.0.0.1/$rke1/g" ~/.kube/rke2-config
export KUBECONFIG=~/.kube/rke2-config
```
Then:

```
watch kubectl get po -A
```

Wait until all pods are `Running` or `Completed`. At that point the RKE2 server (a.k.a. control plane) is functional.

## Install RKE2 workers

### Copy the upstream binaries and this project's install script to the worker VMs

```
for WORKER in $rke2 $rke3; do \
  ssh -i $PRIVKEY root@$WORKER "mkdir /root/rke2-artifacts"; \
  scp -i $PRIVKEY binaries/* scripts/install-rke2* root@$WORKER:/root/rke2-artifacts; \
done
```

### Run the RKE2 installer to install each worker

First get the server token - it is interpolated into the rke2-agent config file in the VM:

```
SERVER_TOKEN=$(ssh -i $PRIVKEY root@$rke1 "cat /var/lib/rancher/rke2/server/node-token") && echo $SERVER_TOKEN
```

Should result in something similar to: `K107717740ad4dbc4fe2b002255ecdb2cef2a1b27a2c2d754908c95d786ba312a2e::server:a160df6b12c776bdcc43f5d171443c39`

Then run the RKE2 install. The two positional params for the script are the IP address of the RKE2 server, and the server token obtained above:

```
for WORKER in $rke2 $rke3; do \
  ssh -i $PRIVKEY root@$WORKER "/root/rke2-artifacts/install-rke2-worker $rke1 $SERVER_TOKEN"; \
done
```

If success, each worker should complete with:

```
Finished.
```

Once both workers have finished, wait for all nodes to be ready:

```
$ kubectl get nodes
NAME   STATUS   ROLES                       AGE     VERSION
rke1   Ready    control-plane,etcd,master   9m8s    v1.21.5+rke2r2
rke2   Ready    <none>                      2m14s   v1.21.5+rke2r2
rke3   Ready    <none>                      27s     v1.21.5+rke2r2
```

Next get all the pods to see that the `DaemonSet`s are running on all the nodes:

```
$ kubectl get po -A -owide | head -n1 && kubectl get po -A -owide | tail -n+2 | sort -k8
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE     IP             NODE   NOMINATED NODE   READINESS GATES
kube-system   helm-install-rke2-ingress-nginx-ksmlq                   0/1     Completed   0          9m11s   10.42.0.2      rke1   <none>           <none>
kube-system   helm-install-rke2-metrics-server-sqkp6                  0/1     Completed   0          9m11s   10.42.0.3      rke1   <none>           <none>
kube-system   rke2-coredns-rke2-coredns-7bb4f446c-qqpjf               1/1     Running     0          8m51s   10.42.0.5      rke1   <none>           <none>
kube-system   rke2-coredns-rke2-coredns-7bb4f446c-sqgvz               1/1     Running     0          2m28s   10.42.0.9      rke1   <none>           <none>
kube-system   rke2-coredns-rke2-coredns-autoscaler-7c58bd5b6c-6lt7g   1/1     Running     0          8m51s   10.42.0.4      rke1   <none>           <none>
kube-system   rke2-metrics-server-5df7d77b5b-jbzrz                    1/1     Running     0          8m34s   10.42.0.6      rke1   <none>           <none>
kube-system   cloud-controller-manager-rke1                           1/1     Running     0          8m47s   192.168.0.31   rke1   <none>           <none>
kube-system   etcd-rke1                                               1/1     Running     0          8m59s   192.168.0.31   rke1   <none>           <none>
kube-system   helm-install-rke2-canal-c7sb7                           0/1     Completed   0          9m11s   192.168.0.31   rke1   <none>           <none>
kube-system   helm-install-rke2-coredns-dgmbt                         0/1     Completed   0          9m11s   192.168.0.31   rke1   <none>           <none>
kube-system   kube-apiserver-rke1                                     1/1     Running     0          8m50s   192.168.0.31   rke1   <none>           <none>
kube-system   kube-controller-manager-rke1                            1/1     Running     0          8m42s   192.168.0.31   rke1   <none>           <none>
kube-system   kube-proxy-rke1                                         1/1     Running     0          8m44s   192.168.0.31   rke1   <none>           <none>
kube-system   kube-scheduler-rke1                                     1/1     Running     0          8m52s   192.168.0.31   rke1   <none>           <none>
kube-system   rke2-canal-d9ld2                                        2/2     Running     0          8m52s   192.168.0.31   rke1   <none>           <none>
kube-system   rke2-ingress-nginx-controller-tvktf                     1/1     Running     0          8m29s   192.168.0.31   rke1   <none>           <none>
kube-system   kube-proxy-rke2                                         1/1     Running     0          2m35s   192.168.0.32   rke2   <none>           <none>
kube-system   rke2-canal-4x497                                        2/2     Running     0          2m36s   192.168.0.32   rke2   <none>           <none>
kube-system   rke2-ingress-nginx-controller-hvktd                     1/1     Running     0          2m16s   192.168.0.32   rke2   <none>           <none>
kube-system   kube-proxy-rke3                                         1/1     Running     0          48s     192.168.0.33   rke3   <none>           <none>
kube-system   rke2-canal-lb7pw                                        2/2     Running     0          49s     192.168.0.33   rke3   <none>           <none>
kube-system   rke2-ingress-nginx-controller-h6pvf                     1/1     Running     0          29s     192.168.0.33   rke3   <none>           <none>
```

You now have a fully functional RKE2 cluster with one control plane node and two workers running on your desktop.
