# Cable Driver Protocol Support Enhancements

## Summary

Submariner today supports three cable drivers: VXLAN, IPSec (tunnel mode) and WireGuard. The
goal of this proposal is to outline additional protocols and enhancements for the Submariner
Cable Driver. These include the additional support of IP-in-IP for unencrypted connections
between clusters and the support of IPSec transport mode.

## Proposal

### IP-in-IP Encapsulation

This enhancement proposes to support IP-in-IP for un-encrypted connection between clusters. A
new IP tunnelling interface `ipip-tunnel` will be added to the active gateway node of each cluster
and it will act as the Tunnel End Point. The traffic which is destined to other clusters will be
forwarded via this interface to the appropriate clusters.

> **_NOTE:_** A single IP Tunnelling interface will be added, and tunnels managed using
`# ip route add encap ip <args>` in order to avoid a point to point connection per tunnel.

#### Design Details

* A new interface "ipip-tunnel" will be added.
* The IP tunnelling interface will be one-to-many, that is a single interface will be used to
connect to all the joined clusters.
* For the "ipip-tunnel" IP, the prefix "242" will be used, followed by the rest of the blocks
from the endpoint private IP address. For example, if the private IP address is 10.2.96.1 the
"ipip-tunnel" IP would be 242.2.96.1.
* The routes will be added to forward all the Service and Pod CIDR traffic to the respective
remote Tunnel IPs.
* Table 100 shall be used to add the routes, which is otherwise used for host network traffic
routes. We do not require any extra rules to handle host network traffic for the IPTun driver.

Example of routeing rules added by the IPTun driver on a gateway node to other nodes

``` bash
ip route add 10.2.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.21 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 100.2.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.21 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 10.3.0.0/16 encap ip id 100 dst 172.18.0.8 via 241.18.0.8 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 100.3.0.0/16 encap ip id 100 dst 172.18.0.8 via 241.18.0.8 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 10.4.0.0/16 encap ip id 100 dst 172.18.0.4 via 241.18.0.4 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 100.4.0.0/16 encap ip id 100 dst 172.18.0.4 via 241.18.0.4 dev ipip0 table 100 metric 100 src 10.1.96.0
```

and the resulting routes on the gateway node:

```bash
[root@cluster1-worker submariner]# ip route show table 100
10.2.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.8 ttl 0 tos 0 via 242.18.0.8 dev ipip-tunnel src 10.1.160.0 metric 100
10.3.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.7 ttl 0 tos 0 via 242.18.0.7 dev ipip-tunnel src 10.1.160.0 metric 100
10.4.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.10 ttl 0 tos 0 via 242.18.0.10 dev ipip-tunnel src 10.1.160.0 metric 100
100.2.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.8 ttl 0 tos 0 via 242.18.0.8 dev ipip-tunnel src 10.1.160.0 metric 100
100.3.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.7 ttl 0 tos 0 via 242.18.0.7 dev ipip-tunnel src 10.1.160.0 metric 100
100.4.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.10 ttl 0 tos 0 via 242.18.0.10 dev ipip-tunnel src 10.1.160.0 metric 100
```

A high level view of the topology is shown in the diagram below:
![IP-in-IP Tunnelled Gateway](./images/ipip_cable.png)

> **_NOTE:_** IP-in-IP has a lower processing and bandwidth overhead than VXLAN.

### IPSec Transport mode

This enhancement proposes to extend the available IPSec offering in Submariner to also provide
transport mode support coupled with the supported overlay protocols (VXLAN, IPinIP...). This
could provide end-to-end security while greatly reducing the performance overhead of tunnel mode
secured tunnels for certain scenarios. In terms of implementation, this would require the addition
of libreswan cable driver configuration parameters to `submariner.io_submariners.yaml`:

```yaml
    ceIPSecMode:
        type: string
    ceIPSecOverlay:
        type: string
```

> **_NOTE:_**
>
> * mode: values can be transport or tunnel (default).
> * overlay: values can be vxlan or iptun (the supported encapsulation drivers in Submariner).

The other option would be to create a new ConfigMap for cable driver configuration and
add this map to the `submariner.io_submariners.yaml` - this would allow for more detailed
configurations to be passed to the cable drivers moving forward without having to extend
the Submariners CRD beyond the addition of the ConfigMap:

```yaml
cableDriverCustomConfig:
  properties:
    configMapName:
      type: string
```

The ConfigMap could be kept relatively simple where keys follow the naming convention:
`ceDriverNameParam`. An example is shown below:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cabledriver-config
data:
  ceCustomConfig: |
    ceIPSecMode: "transport",
    ceIPSecOverlay: "vxlan",
    ceIPSecPfsGroup: ["modp2048"]
    ceIPSecEsp: ["aes_gcm-null"]
```

On the invocation of the `NewLibreswan()` function the configuration params
are read and and establish if the request is for a tunnel or transport based
connection. If the connection is transport based - then the `libreswan` driver
will need to also drive the VXLAN or IPTun Cable Driver as well as the setup, and
teardown of the IPSec connections. The existing VXLAN/IPTun Configuration will be the
same.

An example configuration (that would need to be done by the driver) for a VXLAN over
IPSec Transport connection between two Submariner clusters gateways cluster1-worker
(172.18.0.11) and cluster2-worker (172.18.0.9):

<!-- markdownlint-disable line-length -->
```bash
[root@cluster1-worker]# ipsec whack --psk --encrypt --name submariner-cable-cluster2-172-18-0-9-0 --host 172.18.0.11 --clientproto udp/vxlan --to --host 172.18.0.9 --clientproto udp 
[root@cluster1-worker]# ipsec whack --psk --encrypt  --name submariner-cable-cluster2-172-18-0-9-1 --host 172.18.0.11 --clientproto udp --to --host 172.18.0.9 --clientproto udp/vxlan
[root@cluster1-worker]# ipsec whack --route --name submariner-cable-cluster2-172-18-0-9-0
[root@cluster1-worker]# ipsec whack --route --name submariner-cable-cluster2-172-18-0-9-1
[root@cluster1-worker]# ipsec whack --initiate --asynchronous --name submariner-cable-cluster2-172-18-0-9-0
[root@cluster1-worker]# ipsec whack --initiate --asynchronous --name submariner-cable-cluster2-172-18-0-9-1

[root@cluster2-worker]# ipsec whack --psk --encrypt --name submariner-cable-cluster1-172-18-0-11-0 --host 172.18.0.9 --clientproto udp/vxlan --to --host 172.18.0.11 --clientproto udp 
[root@cluster2-worker]# ipsec whack --psk --encrypt --name submariner-cable-cluster1-172-18-0-11-1 --host 172.18.0.9 --clientproto udp --to --host 172.18.0.11 --clientproto udp/vxlan
[root@cluster2-worker]# ipsec whack --route --name submariner-cable-cluster1-172-18-0-11-0
[root@cluster2-worker]# ipsec whack --route --name submariner-cable-cluster1-172-18-0-11-1
[root@cluster2-worker]# ipsec whack --initiate --asynchronous --name submariner-cable-cluster1-172-18-0-11-0
[root@cluster2-worker]# ipsec whack --initiate --asynchronous --name submariner-cable-cluster1-172-18-0-11-1
```
<!-- markdownlint-enable line-length -->

An example configuration for an IPinIP over IPSec Transport connection between two Submariner
clusters gateways cluster1-worker (172.18.0.11) and cluster2-worker (172.18.0.9):

<!-- markdownlint-disable line-length -->
```bash
[root@cluster1-worker]# ipsec whack --psk --encrypt --name submariner-cable-cluster2-172-18-0-9-0 --host 172.18.0.11 --clientproto ipv4 --to --host 172.18.0.9 --clientproto ipv4 
[root@cluster1-worker]# ipsec whack --route --name submariner-cable-cluster2-172-18-0-9-0
[root@cluster1-worker]# ipsec whack --initiate --asynchronous --name submariner-cable-cluster2-172-18-0-9-0

[root@cluster2-worker]# ipsec whack --psk --encrypt --name submariner-cable-cluster1-172-18-0-11- --host 172.18.0.9 --clientproto ipv4 --to --host 172.18.0.11 --clientproto ipv4 
[root@cluster2-worker]# ipsec whack --route --name submariner-cable-cluster1-172-18-0-11-0
[root@cluster2-worker]# ipsec whack --initiate --asynchronous --name submariner-cable-cluster1-172-18-0-11-0
```
<!-- markdownlint-enable line-length -->

> **_NOTE:_** For VXLAN in transport mode it is recommended to use the default VXLAN port rather than port 4500.