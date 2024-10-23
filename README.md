# ovn-kubernetes-hints
Set of scripts and settings for clean ovn-kubernetes cni installation on the kubernetes (not openshift).

- https://ovn-kubernetes.io/
- https://github.com/ovn-org/ovn-kubernetes
- https://www.ovn.org/en/
- https://ovn-kubernetes.io/design/topology/ (important for topology and architecture)
- https://ovn-kubernetes.io/design/architecture/ (software architecture)
- https://man7.org/linux/man-pages/man5/ovn-nb.5.html (possible tables - network entities inside own)

Suppose the topic deserves own repository, a lot of stuff is just a copy of the 21_ovn in the lab project.


## Important features 

- EgressIP based on the definition (on the namespace, label etc). Important for enterprise environmnets because posibility of cfg firewall rules on higher application sets (applications are in namespaces => ns has egressip). EgressIP must be enabled on the cni.

- Support for keepalived and externalIp. Important for migration from ocp3 to ocp4. Unfortuantelly redhat removes 
command "oc adm ipfailover" and the office documentation just mention the plain deployment of the keepalived via
deployment object:

https://docs.okd.io/latest/networking/configuring-ipfailover.html


## Repository ontent

- ./yaml - object with working ovn object configuration
- ./images - dockerfiles and some set of customized stuff
- ./k8s - rendered kubernetes objects from working cluster
- ./dbs - exports from databases


## Test cluster configuration

Cluster is installed by clean kubeadm command via ansible roles and scripts (look at my lab project and folder 05_k8s). Firewall is disabled and it needs to have more research here to find how to incorporate some ipfilter rules (nftable/iptables) into cni and nodes. Network policy is done by openflow rules.

- cluster is deployed on the local domain lab.syscallx86.com. External dns is provided by freeipa server and all nodes are joined into ipa domain.
- genarally test cluster has one master and 6 nodes
- nodes has only one br-ex interface in 10.1.16.x/24 subnet. Recomendation is two iface in original doc (see github ovn-kubernetes)
- master 10.1.16.11 other nodes 10.1.16.2[1-5], nodes 10.1.16.5[12] has been labeled like gateway and keepalived with external ips has been placed here
- pod network 10.38.0.0/16
- svc network 10.49.0.0/16
- ssl has been disabled (openshift has slighly different configuration where more stuff are placed on one pod => no ssl just unix sockets)
- EgressIp has been enabled

## Basic ovn commands

This description is mainly focused on the north bound database => so the high level network setup of the cni:

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

# kubectl exec --stdin --tty ovnkube-db-84468d897f-764mr -c nb-ovsdb -- /bin/bash
# 
```

The nothbound database contains tables and content of them can be show by command ovn-nbctl list <table name>.

List of tables can be found in man page of the 

https://man7.org/linux/man-pages/man5/ovn-nb.5.html


### List of content of the nb database:

```
# ovn-nbctl show
switch d6dd4a58-cc95-4664-8b98-7aeabb939814 (join)
    port jtor-GR_ovn11.lab.syscallx86.com
        type: router
        router-port: rtoj-GR_ovn11.lab.syscallx86.com
    port jtor-GR_ovn51.lab.syscallx86.com
        type: router
        router-port: rtoj-GR_ovn51.lab.syscallx86.com
    port jtor-GR_ovn16.lab.syscallx86.com
        type: router
        router-port: rtoj-GR_ovn16.lab.syscallx86.com
    port jtor-GR_ovn17.lab.syscallx86.com
        type: router
        router-port: rtoj-GR_ovn17.lab.syscallx86.com
    port jtor-GR_ovn15.lab.syscallx86.com
        type: router
        router-port: rtoj-GR_ovn15.lab.syscallx86.com
    port jtor-ovn_cluster_router
        type: router
        router-port: rtoj-ovn_cluster_router
    port jtor-GR_ovn18.lab.syscallx86.com
        type: router
        router-port: rtoj-GR_ovn18.lab.syscallx86.com
    port jtor-GR_ovn52.lab.syscallx86.com
        type: router
        router-port: rtoj-GR_ovn52.lab.syscallx86.com
switch 6dcf5b88-9ef6-4818-9327-cb70139b372e (ext_ovn17.lab.syscallx86.com)
    port br-ex_ovn17.lab.syscallx86.com
.
.
.
```




## CNI network architecture

Based on the https://ovn-kubernetes.io/design/topology/ ovn has:

- one join switch
- central router ensuring connectivity between pod subnets
- each node has:
    - switch for hosted pods
    - switch for geneve interconnect tunnels
    - router for routing traffic from hosted pods to other nodes:


### Join switch

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


### Ovn cluster router (ovn_cluster_router)

```
# ovn-nbctl list logical_router ovn_cluster_router
_uuid               : 11f24db4-2877-40ba-8beb-87a073964e74
copp                : aad92053-47eb-48a9-b4e2-21cdf8d74137
enabled             : []
external_ids        : {k8s-cluster-router=yes}
load_balancer       : []
load_balancer_group : []
name                : ovn_cluster_router
nat                 : []
options             : {always_learn_from_arp_request="false"}
policies            : [04a02516-905e-4946-9eb3-0b299962bcd9, 1297c63d-a7f8-4fa4-8af9-4e210aea6c16, 31923e3c-31dc-4eff-9117-188ade222f89, 34fea5e5-5c96-4044-b09f-a5acd09c8c81, 5bfe66c4-9b92-48bf-90c2-2a2ce74491da, 6bdbd638-bda9-4d61-9890-341189d79ffc, 6c3dd8c7-42c6-4385-af4b-47cab584754e, 703c7271-a31f-4b12-a084-f5968a02b039, 793fc758-946d-4711-88b9-8a93fd9e616c, 99e43b3b-2f1e-42b8-a122-72c660198d7d, 9da14c68-1616-4e56-8bda-e7059ae77782, b027c249-ef74-4163-b267-18a7717cd5d8]
ports               : [0cb81407-2f13-4500-a797-7466935a8fc4, 1b351699-3b9a-4219-b0cd-1183f8676797, 201b0565-1a2d-4ee7-bfd3-b6e59e7d45c8, 7116454e-6868-4edd-a8b8-0d2a97805cba, 749b5e17-487d-47ce-b4d7-03da832b0d01, 7986f49e-d589-48af-9258-5b99ba033d0c, 7d010b6f-af9f-4b78-acc2-ae1af3727819, f8396b18-e21b-48d7-b7fa-f88301e66bc2]
static_routes       : [004ba08b-6143-4597-b14d-c3b1f69f6eae, 03ce9b7b-fbf4-41b1-84fc-6c3b91cea1d2, 117f4e7b-ddba-4f7e-af31-0304359b4e0e, 19347630-eb8c-41dd-b3f3-ee1b6cce66d2, 2a264cc0-c106-426d-9e28-638bf48f16d7, 3ef297a4-f499-4ed4-b776-7f4577bc0b7c, 6d564b4a-8411-481c-b57e-be0f294bdcbe, 79866ccf-9b46-42fa-b603-41dc55554a28, 91652d92-0315-442e-9618-83c237ab2c5f, 9650fff9-fa05-431f-bf96-63edad847a93, bcd3ae84-726a-41b4-98ab-baacf6d6a28b, c88cac1a-5ed9-4f03-a542-079bdd0adecc, daa25c9f-7672-4798-b338-f2f0a7f515d8, ff9855f0-51b1-41ef-9621-dbb33c49ea6f]
```

Logical router has two types of ports, first all is port to host switch on each nodes and one port to join switch. Router has also static_routes configuration where every host has route to ovn pod network subnet and geneve tunel subnets.

```
# for a in `ovn-nbctl list logical_router ovn_cluster_router | grep static_routes | awk -F\[ '{print $2}' | sed s/\]//g | tr "\," "\n" | sed "s/\s//g"`; do ovn-nbctl list logical_router_static_route $a ; echo "" ; done;
_uuid               : 004ba08b-6143-4597-b14d-c3b1f69f6eae
bfd                 : []
external_ids        : {}
ip_prefix           : "100.64.0.3"
nexthop             : "100.64.0.3"
options             : {}
output_port         : []
policy              : []
route_table         : ""

_uuid               : 19347630-eb8c-41dd-b3f3-ee1b6cce66d2
bfd                 : []
external_ids        : {}
ip_prefix           : "10.38.3.0/24"
nexthop             : "10.38.3.2"
options             : {}
output_port         : []
policy              : src-ip
route_table         : ""
.
.
```

Tbd - see dbs/ovn_cluster_router.out

### Host switches

Host switches are used on the servers for connecting hosted pods and (probably - must be checked) load balancing via services. Kube proxy in case of ovn-kubernetes is not necessary. Names of the host switches are based on the hostname of nodes. 


```
# ovn-nbctl list logical_switch ovn17.lab.syscallx86.com
_uuid               : 6391e29b-635f-41b8-babf-c18440527f08
acls                : [ea07d89e-b511-4cd9-87da-000b21448c8f]
copp                : []
dns_records         : []
external_ids        : {}
forwarding_groups   : []
load_balancer       : [16de93e4-df77-4be9-882e-d1b3c46ca5e5]
load_balancer_group : [b697d980-3f51-49ab-bb2c-6ac52a3cfc50, e5a2c900-9c37-427f-8320-f824b5298567]
name                : ovn17.lab.syscallx86.com
other_config        : {subnet="10.38.3.0/24"}
ports               : [7dba73fa-4a75-4f44-869d-ecb8e0a9d1d5, 81d89c9f-0a9b-41a1-a4b7-831faf85da51, af7a5bef-3e2c-40c3-b724-df0453d25fa6, ba1c2379-c0bf-48bc-9852-624d6016c6de]
qos_rules           : [dfe9eb9a-9320-46c1-80ce-74bdae28e872]
```

#### Host ports and binding to the pods

If we need find appropriate port of the kubernetes pod, we have to first find host where pod is placed


```
$ kubectl get pods -n external-dns -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP          NODE                       NOMINATED NODE   READINESS GATES
external-dns-555764ffff-pqmh4   1/1     Running   31         42d   10.38.3.3   ovn17.lab.syscallx86.com   <none>           <none>
```

Then we can check ports on the host switch and find appropriate record.

```
# for a in `ovn-nbctl list logical_switch ovn17.lab.syscallx86.com | grep ports | awk -F\[ '{print $2}' | sed s/\]//g | tr "\," "\n" | sed "s/\s//g"`; do ovn-nbctl list logical_switch_port $a ; echo "" ; done;
_uuid               : 7dba73fa-4a75-4f44-869d-ecb8e0a9d1d5
addresses           : ["0a:58:0a:26:03:03 10.38.3.3"]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : []
enabled             : []
external_ids        : {namespace=external-dns, pod="true"}
ha_chassis_group    : []
mirror_rules        : []
name                : external-dns_external-dns-555764ffff-pqmh4
options             : {iface-id-ver="4711423e-228e-4deb-b9db-f33b9c4f9452", requested-chassis=ovn17.lab.syscallx86.com}
parent_name         : []
port_security       : ["0a:58:0a:26:03:03 10.38.3.3"]
tag                 : []
tag_request         : []
type                : ""
up                  : true
```



## Possible bugs and quirks of this instalation

- if you enbable externalip is possible to route trafice from outside world to the pod network. Needs to be checked on the ocp4 as well in the future.
