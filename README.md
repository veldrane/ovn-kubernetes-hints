## ovn-kubernetes-hints
Set of scripts and settings for clean ovn-kubernetes cni installation on the kubernetes (not openshift).

- https://ovn-kubernetes.io/
- https://github.com/ovn-org/ovn-kubernetes
- https://www.ovn.org/en/
- https://ovn-kubernetes.io/design/topology/ (important for topology and architecture)
- https://ovn-kubernetes.io/design/architecture/ (software architecture)
- https://man7.org/linux/man-pages/man5/ovn-nb.5.html (possible tables - network entities inside own)

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

- cluster is deplyed on the local domain lab.syscallx86.com. External dns is provided by freeipa server and all nodes are joined into ipa domain.
- genarally test cluster has one master and 6 nodes
- nodes has only one br-ex interface in 10.1.16.x/24 subnet. Recomendation is two iface in original doc (see github ovn-kubernetes)
- master 10.1.16.11 other nodes 10.1.16.2[1-5], nodes 10.1.16.5[12] has been labeled like gateway and keepalived with external ips has been placed here
- pod network 10.38.0.0/16
- svc network 10.49.0.0/16
- ssl has been disabled (openshift has slighly different configuration where more stuff are placed on one pod => no ssl just unix sockets)
- EgressIp has been enabled

### Basic commands

Readme is focused on the north bound database, so high level setup of the cni configuration:

```
# kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE                       NOMINATED NODE   READINESS GATES
ovnkube-db-84468d897f-764mr      2/2     Running   76         56d   10.1.16.11   ovn11.lab.syscallx86.com   <none>           <none>
ovnkube-master-f9c59bd6c-cdpqg   2/2     Running   72         55d   10.1.16.11   ovn11.lab.syscallx86.com   <none>           <none>
ovnkube-node-8zbjr               3/3     Running   108        55d   10.1.16.11   ovn11.lab.syscallx86.com   <none>           <none>
ovnkube-node-9wb5f               3/3     Running   105        55d   10.1.16.15   ovn15.lab.syscallx86.com   <none>           <none>
ovnkube-node-qfsjr               3/3     Running   108        55d   10.1.16.17   ovn17.lab.syscallx86.com   <none>           <none>
ovnkube-node-rcfwk               3/3     Running   105        53d   10.1.16.52   ovn52.lab.syscallx86.com   <none>           <none>
ovnkube-node-rjwwz               3/3     Running   105        55d   10.1.16.16   ovn16.lab.syscallx86.com   <none>           <none>
ovnkube-node-ss9zx               3/3     Running   103        53d   10.1.16.51   ovn51.lab.syscallx86.com   <none>           <none>
ovnkube-node-zzccr               3/3     Running   111        55d   10.1.16.18   ovn18.lab.syscallx86.com   <none>           <none>
```

### Network architecture

Based on the https://ovn-kubernetes.io/design/topology/ ovn has:

- one join switch
- central router ensuring connectivity between pod subnets
- each node has:
    - switch for hosted pods
    - switch for geneve interconnect tunnels
    - router for routing traffic from hosted pods to other nodes:


#### Join switch

```
# ovn-nbctl list logical_switch join
_uuid               : d6dd4a58-cc95-4664-8b98-7aeabb939814
acls                : []
copp                : []
dns_records         : []
external_ids        : {}
forwarding_groups   : []
load_balancer       : []
load_balancer_group : []
name                : join
other_config        : {}
ports               : [10c4f8c0-aef5-44a5-b323-f86ffa5a240d, 8f8134c4-c77f-40a2-a298-a1b18a9e5b82, a4920b7e-f011-4836-9937-aa6dafe6b234, a4d0abd6-daf8-408d-a6c7-9cecd1e1e79c, b8d3bafb-8f9b-41b8-b541-e4e64d1a5353, c30fcd5b-0f9d-49c2-89ba-3f414a266dbb, e6bc5954-1600-45d5-8439-9044242a2591, e6e0d6ea-b51b-4485-9fdd-ffe9f428cdab]
```

We can see 8 ports: 7 ports is connected to the geneve router, one to central router. "Jtor" means "join to router". Counter part on the host is "Rtoj" - means "router to join"

```
# for a in `ovn-nbctl list logical_switch join | grep ports | awk -F\[ '{print $2}' | sed s/\]//g | tr "\," "\n" | sed "s/\s//g"`; do ovn-nbctl list logical_switch_port $a | grep name | grep -v parent ; done;
name                : jtor-GR_ovn11.lab.syscallx86.com
name                : jtor-GR_ovn51.lab.syscallx86.com
name                : jtor-GR_ovn16.lab.syscallx86.com
name                : jtor-GR_ovn17.lab.syscallx86.com
name                : jtor-GR_ovn15.lab.syscallx86.com
name                : jtor-ovn_cluster_router
name                : jtor-GR_ovn18.lab.syscallx86.com
name                : jtor-GR_ovn52.lab.syscallx86.com
```

example of the host port:

```
# ovn-nbctl list logical_switch_port jtor-GR_ovn18.lab.syscallx86.com
_uuid               : e6bc5954-1600-45d5-8439-9044242a2591
addresses           : [router]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : []
enabled             : []
external_ids        : {}
ha_chassis_group    : []
mirror_rules        : []
name                : jtor-GR_ovn18.lab.syscallx86.com
options             : {router-port=rtoj-GR_ovn18.lab.syscallx86.com}
parent_name         : []
port_security       : []
tag                 : []
tag_request         : []
type                : router
up                  : true
```




### Possible bugs and quirks of this instalation

- if you enbable externalip is possible to route trafice from outside world to the pod network. Needs to be checked on the ocp4 as well in the future.
