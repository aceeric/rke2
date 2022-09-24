# Three node desktop RKE2 / Rancher cluster

This scenario creates a three-node desktop RKE2 cluster consisting of three VMs: one control plane node, and two workers. This is a disconnected install. You copy the install files to the VMs and run the install script in the VM. The control plane node also gets Rancher installed by the install script.

The RKE2 air-gapped install instructions are at: https://docs.rke2.io/install/airgap/

## Pre-requisites

You need:

1. Three VMs with CentOS installed. I've only tested with CentOS Stream. Other variants would likely work fine.
2. An ssh private key matching the corresponding public key in each VM
3. Binaries staged on your local workstation as described in the sections below

Since this is an air-gapped install, you need three sets of artifacts. These artifacts only have to be downloaded to your workstation once and be be repeatedly used to generate RKE2 clusters:

1. The RKE2 binaries
2. Rancher and related Helm charts - and Helm itself
3. Rancher and related OCI images

These are now discussed:

### Download RKE2 binaries

This version of the project uses RKE2 v1.23.9 because v1.24.3 was incompatible with the v3.9.2 Rancher Helm chart. The RKE2 binaries go into the `rke2-artifacts` directory:

```
$ export RKE2VER=v1.23.9 && export REL=rke2r1

$ curl -sfL https://github.com/rancher/rke2/releases/download/$RKE2VER%2B$REL/rke2-images.linux-amd64.tar.zst\
  -o ./rke2-artifacts/rke2-images.linux-amd64.tar.zst &&\
  curl -sfL https://github.com/rancher/rke2/releases/download/$RKE2VER%2B$REL/rke2.linux-amd64.tar.gz\
  -o ./rke2-artifacts/rke2.linux-amd64.tar.gz &&\
  curl -sfL https://github.com/rancher/rke2/releases/download/$RKE2VER%2B$REL/sha256sum-amd64.txt\
  -o ./rke2-artifacts/sha256sum-amd64.txt &&\
  curl -sfL https://get.rke2.io\
  -o ./rke2-artifacts/install.sh
```

_Note: %2B in the URLs above is an encoded plus sign (+)._

### Download Rancher binaries

Since this is an air-gapped install, download the Rancher / Cert Manager binaries to your local workstation. Subsequently, they will be copied into the VM where the RKE2 server is installed. Rancher is installed using Helm, so get the Helm installer and the Rancher / Cert Manager Helm charts. These artifacts go into the `rancher-artifacts` directory:

```
$ curl -sfL https://get.helm.sh/helm-v3.9.2-linux-386.tar.gz\
  -o ./rancher-artifacts/helm-v3.9.2-linux-386.tar.gz &&\
  curl -sfL https://releases.rancher.com/server-charts/stable/rancher-2.6.6.tgz\
  -o ./rancher-artifacts/rancher-2.6.6.tgz &&\
  curl -sfL https://charts.jetstack.io/charts/cert-manager-v1.7.1.tgz\
  -o ./rancher-artifacts/cert-manager-v1.7.1.tgz &&\
  curl -sfL https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml\
  -o rancher-artifacts/cert-manager-crd.yaml
```

### Download Rancher / Cert Manager images

Since this is an air-gapped install, you must obtain the OCI images needed by the Rancher / Cert Manager Helm install artifacts. There are two ways to accomplish this provided by this project. See `hack/gen-rancher-image-tars-ctr`. This script can be run after Rancher is installed in a non-airgapped manner to export the Rancher images from the containerd cache. See `hack/gen-rancher-image-tars-podman` for an alternate way to do it using podman. Either way - the list of Rancher / Cert Manager images was devised by first installing RKE2, then Rancher, and comparing the new images in the containerd cache introduced by the Rancher / Cert Manager install.

Regardless of the method you chose, place the images into the `rancher-images` folder, which is where the remainder of this README expects them.

## Create the VMs

This project uses scripts developed as part of the https://github.com/aceeric/desktop-kubernetes project to create VirtualBox VMs. _Desktop Kubernetes_ stands up a three-VM k8s cluster using VirtualBox, the CentOS Stream ISO, and the various upstream Kubernetes components. The _Desktop Kubernetes_ project also has functionality that simplifies creating VirtualBox VMs. So this project uses that functionality.

You can create the VMs however you want. The way I created then with _Desktop Kubernetes_ is shown for documentation purposes only.

### Use Desktop Kubernetes to create the VMs

This section assumes you've git cloned _Desktop Kubernetes_ into `~/projects/desktop-kubernetes` and that is your current working directory. If you git cloned differently, adjust any instructions.

First create a template VM. This VM will have CentOS Stream and Virtual Box Guest Additions. This example uses VirtualBox bridge networking via the `--host-network-interface` arg. I had trouble with the host-only networking in VBox and am still researching a resolution so for now, the VMs must be configured with bridge networking or the RKE2 agent doesn't start. The value for the host network interface is unique to each workstation. Use `ifconfig` to get your primary network interface name.

```
hack/scripts/do-create-template-vm --host-network-interface enp0s31f6 rke2-template /sdb1/virtualboxvms
```

The above statement creates a template VM named `rke2-template`. Next create three clones from the template:

```
for clone in rke1 rke2 rke3; do \
  hack/scripts/do-create-clone-vm --host-network-interface enp0s31f6 rke2-template $clone /sdb1/virtualboxvms; \
done
```

You now have three Virtual Box VMs running named `rke1`, `rke2`, and `rke3`. Get the IP addresses for the VMs:

```
$ for host in rke1 rke2 rke3; do echo export $host=$(scripts/helpers/get-vm-ip $host); done
export rke1=192.168.0.31
export rke2=192.168.0.32
export rke3=192.168.0.33
```

# In this (rke2) Project

From here forward, the instructions assume you've git cloned https://github.com/aceeric/rke2 and the repo is your current working directory. E.g. `cd ~/projects/rke2`.

Above when you generated a VM using _Desktop Kubernetes_ it created an ed25519 SSH key pair and places the public key into each VM and the private key in `~/projects/desktop-kubernetes/generated/kickstart/id_ed25519`. You need that SSH key to access the VMs here. So create an environment variable to make the ssh command simpler:

```
export PRIVKEY=~/projects/desktop-kubernetes/generated/kickstart/id_ed25519
```

Set environment variables for the VMs using the values above

```
export rke1=192.168.0.31
export rke2=192.168.0.32
export rke3=192.168.0.33
```

## Install RKE2 server

This step assumes you've downloaded all the components needed to install RKE2 and Rancher as described above.

### Copy all server-related artifacts to the server VM

```
ssh -i $PRIVKEY root@$rke1 "mkdir -p /root/rke2-artifacts && mkdir -p /root/rancher-artifacts && mkdir -p /root/rancher-images"
scp -i $PRIVKEY rke2-artifacts/* scripts/install-rke2* root@$rke1:/root/rke2-artifacts
scp -i $PRIVKEY rancher-artifacts/* root@$rke1:/root/rancher-artifacts
scp -i $PRIVKEY rancher-images/* root@$rke1:/root/rancher-images
```

### Run the RKE2 installer to install the control plane and Rancher

```
ssh -i $PRIVKEY root@$rke1 /root/rke2-artifacts/install-rke2-server
```

If success you should see this:

```
To access the RKE2 cluster from outside the VM using kubectl:

export PRIVKEY=<path to the private key matching the public key in the VM>
scp -i $PRIVKEY root@192.168.0.31:/etc/rancher/rke2/rke2.yaml ~/.kube/rke2-config
sed -i 's/127.0.0.1/192.168.0.31/g' ~/.kube/rke2-config
export KUBECONFIG=~/.kube/rke2-config

To access the Rancher UI from you browser:

Add an entry to your etc hosts: '192.168.0.31 rke1' then access
https://rke1 from your browser. Remember to disable all tracking
protection in your browser first.
```

On my system, which has the following CPU config - Intel i7-8700 CPU @ 3.20GHz x 12 - generating the RKE2 server with Rancher in the VM takes about five minutes.

Execute the following commands to configure your workstation to access the cluster VMs - which at this point consists of a single control plane node:

```
scp -i $PRIVKEY root@$rke1:/etc/rancher/rke2/rke2.yaml ~/.kube/rke2-config
sed -i "s/127.0.0.1/$rke1/g" ~/.kube/rke2-config
export KUBECONFIG=~/.kube/rke2-config
```
Then:

```
kubectl get po -A
```

All pods should be `Running` or `Completed`. At this point, the RKE2 server (a.k.a. control plane) is functional.

## Install RKE2 workers

### Copy the upstream binaries and this project's install script to the worker VMs

This step also copies the server kubeconfig into the worker so `kubectl` in the workers can be used to interact with the cluster.

```
for WORKER in $rke2 $rke3; do \
  ssh -i $PRIVKEY root@$WORKER "mkdir -p /root/rke2-artifacts && mkdir -p /root/.kube"; \
  scp -3 -i $PRIVKEY root@$rke1:/etc/rancher/rke2/rke2.yaml root@$WORKER:/root/.kube/config; \
  scp -i $PRIVKEY rke2-artifacts/* scripts/install-rke2* root@$WORKER:/root/rke2-artifacts; \
done
```

### Run the RKE2 installer to install each worker

First get the RKE2 server token - it is passed as an arg to the worker installation script and is needed to join a worker to the cluster.

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
done installing rke2 worker
```

Once both workers have finished, wait for all nodes to be ready:

```
$ kubectl get nodes
NAME   STATUS   ROLES                       AGE     VERSION
rke1   Ready    control-plane,etcd,master   9m8s    v1.23.9+rke2r1
rke2   Ready    <none>                      2m14s   v1.23.9+rke2r1
rke3   Ready    <none>                      27s     v1.23.9+rke2r1
```

Next get all the pods to confirm that all the nodes are running pods:

```
$ (echo NAMESPACE NAME STATUS NODE && kubectl get po -A --no-headers\
  -o custom-columns=NS:metadata.namespace,NAME:metadata.name,STATUS:status.phase,NODE:spec.nodeName | sort -k4)\
  | column -t
NAMESPACE                  NAME                                                   STATUS     NODE
cattle-system              helm-operation-566x8                                   Failed     rke1
cattle-system              helm-operation-5v4hm                                   Failed     rke1
cattle-system              helm-operation-br8jw                                   Failed     rke1
cattle-fleet-local-system  fleet-agent-699b5fb945-6frhc                           Running    rke1
cattle-fleet-system        fleet-controller-784d6fbcd8-ptfv8                      Running    rke1
cattle-fleet-system        gitjob-6b977748fc-7ph4v                                Running    rke1
cattle-system              helm-operation-ckwd6                                   Running    rke1
cattle-system              helm-operation-qv76z                                   Running    rke1
cattle-system              helm-operation-r9rhn                                   Running    rke1
cattle-system              helm-operation-xk66c                                   Running    rke1
cattle-system              rancher-c7fd85b48-lpj9j                                Running    rke1
cert-manager               cert-manager-76d44b459c-wpp6f                          Running    rke1
cert-manager               cert-manager-cainjector-9b679cc6-vn28r                 Running    rke1
cert-manager               cert-manager-webhook-57c994b6b9-8f2f9                  Running    rke1
kube-system                cloud-controller-manager-rke1                          Running    rke1
kube-system                etcd-rke1                                              Running    rke1
kube-system                kube-apiserver-rke1                                    Running    rke1
kube-system                kube-controller-manager-rke1                           Running    rke1
kube-system                kube-proxy-rke1                                        Running    rke1
kube-system                kube-scheduler-rke1                                    Running    rke1
kube-system                rke2-canal-7nqd5                                       Running    rke1
kube-system                rke2-coredns-rke2-coredns-545d64676-kcsqd              Running    rke1
kube-system                rke2-coredns-rke2-coredns-autoscaler-5dd676f5c7-kmz29  Running    rke1
kube-system                rke2-ingress-nginx-controller-cjh6f                    Running    rke1
kube-system                rke2-metrics-server-6564db4569-fskv5                   Running    rke1
cattle-system              helm-operation-8m5bt                                   Succeeded  rke1
cattle-system              helm-operation-hffr8                                   Succeeded  rke1
cattle-system              helm-operation-klk5f                                   Succeeded  rke1
cattle-system              helm-operation-rjghv                                   Succeeded  rke1
cert-manager               cert-manager-startupapicheck-6hxhj                     Succeeded  rke1
kube-system                helm-install-rke2-canal-nclbp                          Succeeded  rke1
kube-system                helm-install-rke2-coredns-ljmnf                        Succeeded  rke1
kube-system                helm-install-rke2-ingress-nginx-ss56t                  Succeeded  rke1
kube-system                helm-install-rke2-metrics-server-vn9p5                 Succeeded  rke1
cattle-system              helm-operation-4n7sb                                   Running    rke2
cattle-system              helm-operation-fgzsq                                   Running    rke2
cattle-system              helm-operation-pkfcs                                   Running    rke2
cattle-system              rancher-webhook-5b65595df9-pz524                       Running    rke2
kube-system                kube-proxy-rke2                                        Running    rke2
kube-system                rke2-canal-w7m78                                       Running    rke2
kube-system                rke2-coredns-rke2-coredns-545d64676-dqjxt              Running    rke2
kube-system                rke2-ingress-nginx-controller-4gqh4                    Running    rke2
cattle-system              helm-operation-dmf8z                                   Running    rke3
cattle-system              helm-operation-hxzn2                                   Running    rke3
cattle-system              helm-operation-jw64b                                   Running    rke3
cattle-system              helm-operation-s7jgj                                   Running    rke3
kube-system                kube-proxy-rke3                                        Running    rke3
kube-system                rke2-canal-f9dbz                                       Running    rke3
kube-system                rke2-ingress-nginx-controller-vpfvm                    Running    rke3
```

You now have a fully functional RKE2 cluster with one control plane node and two workers running on your desktop, with Rancher installed on the control plane.

You can access the Rancher UI by adding an entry into your /etc/hosts like:

```
192.168.0.31 rke1
```

Then in your browser, navigate to https://rke1. Make sure to disable an browser tracking protection. The install scripts set the bootstrap admin password to `admin`. You'll be required to change it on initial login to the Rancher UI.
