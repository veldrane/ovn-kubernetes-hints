## ovn-kubernetes-hints
Set of scripts and settings for clean ovn-kubernetes cni installation on the kubernetes (not openshift).

- https://ovn-kubernetes.io/
- https://github.com/ovn-org/ovn-kubernetes
- https://www.ovn.org/en/

Suppose the topic deserves own repository, a lot of stuff is just a copy of the 21_ovn in the lab project.


### Important features 

- EgressIP based on the definition (on the namespace, label etc). Important for enterprise environmnets because posibility of cfg firewall rules on higher application sets (applications are in namespaces => ns has egressip). EgressIP must be enabled on the cni.

- Support for keepalived and externalIp. Important for migration from ocp3 to ocp4. Unfortuantelly redhat removes 
command "oc adm ipfailover" and the office documentation just mention the plain deployment of the keepalived via
deployment object:

https://docs.okd.io/latest/networking/configuring-ipfailover.html


### Content

- ./yaml - object with working ovn object configuration
- ./images - dockerfiles and some set of customized stuff
- ./k8s - rendered kubernetes objects from working cluster


### Configuration

Cluster is installed by clean kubeadm command via ansible roles and scripts (look at my lab direcotry). Firewall is disabled and it needs to be more research here to find how to incorporate some ipfilter rules (nftable/iptables) into cni and nodes. Network policy is done by openflow rules.

- nodes has only one br-ex interface in 10.1.16.x/24 subnet. Recomendation is two iface in original doc (see github ovn-kubernetes)
- master 10.1.16.11 other nodes 10.1.16.2[1-5], nodes 10.1.16.5[12] has been labeled like gateway and keepalived with external ips has been placed here
- pod network 10.38.0.0/16
- svc network 10.49.0.0/16
- ssl has been disabled (openshift has slighly different configuration where more stuff are placed on one pod => no ssl just unix sockets)
- EgressIp has been enabled


### Posible bugs and quirks of this instalation

- if you enbable externalip is possible to route trafice from outside world to the pod network. Needs to be checked on the ocp4 as well in the future.
