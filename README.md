# RKE2 installation scenarios

This repo demonstrates various scenarios for installing RKE2. All work assumes you're running on a Linux desktop. All development and testing was done on Ubuntu 20.04.4 LTS.

## Scenarios so far

1. `scenarios/vbox-desktop` Installs a three-VM RKE2/Rancher cluster on desktop VirtualBox VMs with Rancher running on the control plane node. (Airgapped install.)
2. `scenarios/connected` Installs a single-node, single-VM RKE2/Rancher cluster into a VM with internet connectivity. (Connected install.)
