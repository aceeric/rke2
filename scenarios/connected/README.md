# Single node RKE2/Rancher cluster

This scenario creates a single-node RKE2 cluster on one VM that has  internet access. See `scenarios/vbox-desktop/README.md` for instructions on provisioning a VM using VirtualBox.

_Last tested on 10-Oct-2022._

## Steps

1. Provision a VM.
2. Copy the script `scripts/install-rke2-rancher-connected-single-node` into the VM home directory.
3. Run the script as root.
4. Add an entry in your `/etc/hosts` for the VM name because the Rancher UI is accessed via an Nginx ingress with TLS and the VM hostname configured into the ingress by Rancher.
5. Access the Rancher UI using the VM hostname. e.g.: https://myvmname. The bootstrap administrator password is _admin_, which you must change on initial login to the Rancher UI.
