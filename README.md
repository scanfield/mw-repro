This repo contains a set of scripts to reproduce an issue somewhere
between GKE, K8S, and PD's multi writer features.

The goal is to have 2 K8S pods able to read and write from the same PD
as a block device.

This works as expected with 2 "raw" GCE VMs, but once in GKE you get
strange behavior. Writes from pod A are not visible in pod B unless
you re-attach the disk. This can be achieved by restarting pod B.
