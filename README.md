## ovn-kubernetes-hints
Set of scripts and settings for clean ovn-kubernetes cni installation on the kubernetes (not openshift).

- https://ovn-kubernetes.io/
- https://github.com/ovn-org/ovn-kubernetes
- https://www.ovn.org/en/

### Content

- ./yaml - object with working ovn object configuration


### Configuration

Cluster is installed by clean kubeadm command via ansible roles and scripts (look at my lab direcotry). Firewall is disabled and it needs to be more research here to find how to incorporate some ipfilter rules (nftable/iptables) into cni and nodes. Network policy is done by openflow rules.


### Important features 

- EgressIP based on the definition (on the namespace, label etc). Important for enterprise environmnets because posibility of cfg firewall rules on higher application sets (applications are in namespaces => ns has egressip). EgressIP must be enabled on the cni.

- Support for keepalived and externalIp. Important for migration from ocp3 to ocp4. Unfortuantelly redhat removes 
command "oc adm ipfailover" and the office documentation just mention the plain deployment of the keepalived via
deployment object:

https://docs.okd.io/latest/networking/configuring-ipfailover.html


### Posible bugs and quirks of this instalation


