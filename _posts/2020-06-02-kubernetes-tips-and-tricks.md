---
layout: post
title: Kubernetes Tips and Tricks
categories:
header_image: "/img/k8s-tips-tricks.jpg"
header_permalink: "https://unsplash.com/photos/Esq0ovRY-Zs"
summary: "Some commands you should never use, some you should always use"

---

# {{ page.title }}

A few tips and tricks I've come across. Starting off small!

## Export Cluster Config from Kubeconfig

Say you have a whole bunch of clusters set in your kubeconfig file and you want to extract one. Just one.

Set your config to that cluster (maybe use kubectx) and do:

```
kubectl config view --minify --raw > cluster.kubeconfig
```

Boom!

## Merge Kube Configs

Copy your backup, set an env var pointing to the backup config and the new standalone file, and use config view with the flatten option to produce a new, merged, config file, and finally copy that file back to ~/.kube/config.

```
$ cp ~/.kube/config ./config-backup 
$ KUBECONFIG=./config-backup:./new-standalone.kubeconfig kubectl config view --flatten > new-kube-config
$ cp new-kube-config ~/.kube/config 
```

## Writing an Operator in Shell!

See the [shell operator](https://github.com/flant/shell-operator). Good times!

## Troubleshoot DNS

See this [k8s doc](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

```
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```

Make sure you are in the default namespace.

Run a dig command from the pod.

```
$ kubectl exec -i -t dnsutils -- dig +short google.com
172.217.165.14
```