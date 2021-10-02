This repo contains a set of scripts to reproduce an issue somewhere
between GKE, K8S, and PD's multi writer features.

The goal is to have 2 K8S pods able to read and write from the same PD
as a block device.

This works as expected with 2 "raw" GCE VMs (and writes are
~immediately visible), but once in GKE you get strange behavior.
Writes from pod A are not visible in pod B unless you re-attach the
disk. This can be achieved by restarting pod B.

You might have to adjust the commands below, but they work for me.

#### First step, create a new GCP project:
`gcloud projects create --name="MW Repro"`

#### set it in your config
`gcloud config set project $PROJECT`
`gcloud config set compute/zone us-central1-c`

#### At this point I had to go into the UI to enable billing / enable the API.

#### now invent a gke cluster:
`gcloud container clusters create mw-cluster-0 --machine-type=n2-standard-2 --num-nodes=2`

#### we need one of these magic disks; the PVC tooling can't generate one on its own yet (can't do the --multi-writer flag)
`gcloud beta compute disks create mw-disk-0 --size=100GB --type=pd-ssd --multi-writer`

#### now invent a PV and PVC based on this disk:
`kubectl apply -f disks.yaml`

#### Now invent a couple of pods:
`kubectl apply -f pods.yaml`

#### Wait a sec for the pods to be up....

#### Now let's start writing to the PD
`kubectl exec $pod0 -- bash -c "hostname > /mw/identity.txt; dd if=/mw/identity.txt of=/mw/block bs=4096 oflag=direct conv=sync"`

#### can we see this write on pod0? (answer should be yes)
`kubectl exec $pod0 -- dd if=/mw/block bs=4096 count=1`

#### Can we see this in the other pod? (Nope)
`kubectl exec $pod1 -- dd if=/mw/block bs=4096 count=1`

#### Okay, now let's kill pod0 and see if that shakes something free:
`kubectl delete pod $pod0`
#### k8s invents pod 2.

#### This read still sees the data, so we know the write was persisted.
`kubectl exec $pod2 -- dd if=/mw/block bs=4096 count=1`

#### But pod1 stil doesn't see any data:
`kubectl exec $pod1 -- dd if=/mw/block bs=4096 count=1`

#### I've waited at this point for hours, pod1 never learns about the write.

#### Finally, kill pod1
`kubectl delete pod $pod1`

#### Finally pod3 sees the write from pod0
`kubectl exec $pod3 -- dd if=/mw/block bs=4096 count=1`
