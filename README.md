# Lessons Learned: Kubernetes, The Hard Way, March 2019

Takeaway: installing Kubernetes "the hard way" is hard because of the constant evolving of the project.

Here, we do bunch of "claims". Claims are documentation told somewhere. Then, "errors" are what actually should be done as of __March, 2019__.

- claim: [use kube-dns](https://coreos.com/kubernetes/docs/1.6.1/deploy-addons.html)
- error: [as of Kubernetes v1.12, CoreDNS is the recommended DNS Server, replacing kube-dns](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)

---

- claim: use [Kubernetes version from quay.io/coreos/hyperkube](https://coreos.com/kubernetes/docs/1.6.1/deploy-master.html)
- error: versions beyond 1.9.6 are not available, use [gcr.io/google-containers/hyperkube-amd64](https://console.cloud.google.com/gcr/images/google-containers/GLOBAL/hyperkube-amd64?gcrImageListsize=30) instead, [to do this, you need to specify --insecure-options=image flag to rkt](https://github.com/coreos/bugs/issues/2470#issuecomment-407088776), which [looks like this](https://github.com/toldjuuso/kubernetes-the-march-2019-way/blob/ea6296622ac872a3dcf929e9d4f6a19e4c52b099/filesystems/master/etc/systemd/system/kubelet.service#L6)

---

- claim: [use --api-servers flag with kubelet](https://coreos.com/kubernetes/docs/1.6.1/deploy-master.html)
- error: [--api-servers is deprecated](https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/158), use [--kubeconfig flag instead](https://github.com/toldjuuso/kubernetes-the-march-2019-way/blob/ea6296622ac872a3dcf929e9d4f6a19e4c52b099/filesystems/master/etc/systemd/system/kubelet.service#L26), which [maps to a file like this](https://github.com/toldjuuso/kubernetes-the-march-2019-way/blob/1d67624905d9eab42d2c5261f27b7bddf2952438/filesystems/master/etc/kubernetes/master.kubeconfig)

---

- assumption: removing [--register-schedulable flag](https://coreos.com/kubernetes/docs/1.6.1/deploy-master.html) will make the master schedulable
- error: commenting it out has no effect, the node has [to be patched instead](https://stackoverflow.com/a/52775256/2464828)

---

- assumption: dropping the self-signed TLS assets would start working after running `sudo systemctl daemon-reload`
- error: you still get "Unable to connect to the server: x509: certificate signed by unknown authority" error on kubelet-wrapper logs
- therapy: [add `--client-ca-file` to kube-apiserver and kubelet](https://s.itho.me/day/2017/k8s/1020-1100%20All%20The%20Troubles%20You%20Get%20Into%20When%20Setting%20Up%20a%20Production-ready%20Kubernetes%20Cluster.pdf), then [`‌openssl s_client -connect ${master-host}:443` will still show that tsl is not applied](https://twitter.com/toldjuuso/status/1102626486517927936) -> you need to restart the host :)

---

- claim: for advanced config, [use etcd with discovery](https://coreos.com/os/docs/latest/installing-to-disk.html)
- error: as the nodes are reachable in LAN, then it simpler to use [named initial cluster](https://github.com/toldjuuso/kubernetes-the-march-2019-way/blob/53f373d7008d23c45dac12bf35676ecbad93124e/provisioning/master.yaml#L26)

---

- assumption: containers can reach google.com
- error: not with my router [(and seemingly others)](https://github.com/coreos/flannel/issues/983#issuecomment-383680337), [add upstreamNameservers to CoreDNS configuration to avoid issues](https://github.com/toldjuuso/kubernetes-the-march-2019-way/commit/beec22730e19d0fe87aaf473819b3940ab385a61)

---

- claim: [according to CoreOS documentation](https://coreos.com/kubernetes/docs/1.6.1/deploy-workers.html), the worker kube-proxy configuration takes in master host without protocol prefix
- error: [CoreOS documentation is wrong](https://stackoverflow.com/a/44025007/2464828), you must specify https protocol prefix, [like this](https://github.com/toldjuuso/kubernetes-the-march-2019-way/commit/042b5636be961709a9478bb9b5b4eb55e226a468)

---

- error: `no route to host` with pod-to-pod and pod-to-apiserver connections
- therapy: [flannel default configuration creates a conflicting `cni0` and `docker0` interfaces](https://github.com/coreos/flannel/issues/1013) on the control-plane node, of which the `cni0` subnet [has to be refreshed by running `ip link delete cni0`](https://github.com/kubernetes/kubernetes/issues/39557#issuecomment-457839765), after which `sudo systemctl restart kubelet` should fix the connection problem -- further, this change seems to persist on reboots

---

bonuses:

- assumption: computers run on the same time
- error: Arch Linux runs 1 minute in the future, [causing etcd consensus to fail](https://github.com/etcd-io/etcd/issues/7051)
- therapy: install and enable NTP client on Arch Linux
