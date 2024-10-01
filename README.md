# BGPunnumbered

## Requirements
- __[Containerlab](https://containerlab.dev)__

## Summary

This demo is intended to show how quickly and easily an engineer can deploy an eBGP unnumbered underlay.

>This was adapted and simplified from [this example tutorial](https://learn.srlinux.dev/tutorials/l3evpn/rt5-only/underlay/)

[RFC 7938](https://datatracker.ietf.org/doc/html/rfc7938) explains the benefits of using eBGP as the underlay routing protocol in lieu of an IGP such as ISIS or OSPF. 
 - __Flexible Policy:__ BGP is well known for being able to match many different attributes to make traffic engineering decisions.
 - __Smaller failure domain:__ If a link fails on a network running a link-state IGP, all devices in the network need to update their link-state database and then run their SPF algorithm, but in a network running eBGP, just the NH is withdrawn from directly attached devices.
 - __Scalable:__ eBGP has been proven to be an extremely scalable routing protocol and can easily scale from the smallest fabrics all the way to the largest with no impact on performance.

The drawback is that configuring eBGP as the underlay has always been a very manual procedure, but using [IPv6 Link Local Addresses](https://en.wikipedia.org/wiki/Link-local_address) in combination with [RFC 8950](https://datatracker.ietf.org/doc/html/rfc8950) (IPv4 NRLI with IPv6 NH), we can drastically reduce the amount of manual configuration while making the configuration easily replicable.


## Topology

```
                                                                 
              ┌───────────┐           ┌───────────┐              
              │    sp1    │           │    sp2    │              
              │           │           │           │              
              │   65000   │           │   65000   │              
              └──┬──┬──┬──┘           └──┬──┬──┬──┘              
                1│ 2│ 3│                1│ 2│ 3│                 
     ┌───────────┘  │  │                 │  │  └───────────┐     
     │              │  └─────────────────┼──┼────────┐     │     
     │     ┌────────┼────────────────────┘  │        │     │     
     │1   2│        └────────┐1   2┌────────┘        │1   2│     
  ┌──┴─────┴──┐           ┌──┴─────┴──┐           ┌──┴─────┴──┐  
  │    l1     │           │    l2     │           │    l3     │  
  │           │           │           │           │           │  
  │   65101   │           │   65102   │           │   65103   │  
  └────┬──────┘           └─────┬─────┘           └─────┬─────┘  
       │3                       │3                      │3       
       │                        │                       │        
       └─────────┐  ┌───────────┘                       │        
                 │  │                                   │        
             eth1│  │eth2                           eth1│        
              ┌──┴──┴───┐                          ┌────┴────┐   
              │         │                          │         │   
              │ client1 │                          │ client2 │   
              └─────────┘                          └─────────┘   
```

# How to start the lab

1. Clone this repository
2. `cd` into the directory where the repository was cloned
3. Run the following to start the lab:

> `sudo clab deploy -t topo.yml -c`

_the `-c` flag is the "reconfigure" option; when starting a topology with this flag, it will always reset the topology to the base using whatever options/configurations are included in the topology file_

Once the topology comes up, you will have an IPv6 underlay using link-local addressing with dynamic eBGP neighbors advertising their IPv4 loopbacks over the IPv6 sessions.

# Configuration Steps

> These examples are taken from the [li.cfg](configs/l1.cfg) file - look through all of the config files in the [configs directory](configs/)

## Physical Interfaces

Fabric Uplinks:

Enable the interface, enable IPv6 and then be sure to enable RA on the interface

```
set / interface ethernet-1/1 admin-state enable
set / interface ethernet-1/1 subinterface 1 admin-state enable
set / interface ethernet-1/1 subinterface 1 ipv6 admin-state enable
set / interface ethernet-1/1 subinterface 1 ipv6 router-advertisement router-role admin-state enable

set / interface ethernet-1/2 admin-state enable
set / interface ethernet-1/2 subinterface 1 admin-state enable
set / interface ethernet-1/2 subinterface 1 ipv6 admin-state enable
set / interface ethernet-1/2 subinterface 1 ipv6 router-advertisement router-role admin-state enable
```

## Loopback Interfaces

Enable the interface and the subinterface and apply the ipv4 /32 address

```
set / interface system0 admin-state enable
set / interface system0 subinterface 0 admin-state enable
set / interface system0 subinterface 0 ipv4 admin-state enable
set / interface system0 subinterface 0 ipv4 address 10.0.0.1/32
```

## Put Subinterfaces In Default Network-Instance

```
set / network-instance default interface system0.0
set / network-instance default interface ethernet-1/1.1
set / network-instance default interface ethernet-1/2.1
set / network-instance default interface ethernet-1/3.1
```

## Allow IPv4 Traffic Over IPv6 Interface

In order to allow ipv4 traffic to be sent/received over ipv6-only interfaces, we need to tell the system to allow it

```
set / network-instance default ip-forwarding receive-ipv4-check false
```

## eBGP Unnumbered

Configure policy: Here we are just creating a very basic policy that allows any ipv4 /32 loopback address (we will use this for both importing and exporting)
```
set / routing-policy prefix-set system-loopbacks prefix 10.0.0.0/8 mask-length-range 32..32

set / routing-policy policy system-loopbacks-policy statement 1 match prefix-set system-loopbacks
set / routing-policy policy system-loopbacks-policy statement 1 action policy-result accept
```

Here are the general BGP configurations required: Enable BGP, set the ASN, set the router-id and enable the ipv4-unicast AF
```
set / network-instance default protocols bgp admin-state enable
set / network-instance default protocols bgp autonomous-system 65101
set / network-instance default protocols bgp router-id 10.0.0.1
set / network-instance default protocols bgp afi-safi ipv4-unicast admin-state enable
```

Configure the BGP group: Enable the group, apply import/export policy, add local-as and enable ipv4-unicast AF
```
set / network-instance default protocols bgp group underlay admin-state enable
set / network-instance default protocols bgp group underlay export-policy [ system-loopbacks-policy ]
set / network-instance default protocols bgp group underlay import-policy [ system-loopbacks-policy ]
set / network-instance default protocols bgp group underlay afi-safi ipv4-unicast admin-state enable
set / network-instance default protocols bgp group underlay local-as as-number 65101
```

Finally, configure the dynamic peers: Note the range of AS allowed for dynamic peering - this is on the l1, so it is only peering with the spine layer which is 65000 - this range does not have to be this restrictive
```
set / network-instance default protocols bgp dynamic-neighbors interface ethernet-1/1.1 peer-group underlay allowed-peer-as [ 65000..65000 ]
set / network-instance default protocols bgp dynamic-neighbors interface ethernet-1/2.1 peer-group underlay allowed-peer-as [ 65000..65000 ]
```

# Next Steps

Standing up this lab will establish the underlay. The next step will be to add the overlay - it is recommended to use iBGP with EVPN VXLAN. 

Refer back to [learn.srlinux.dev](https://learn.srlinux.dev/tutorials/) and try to implement each of these:
- __[L2 EVPN](https://learn.srlinux.dev/tutorials/l2evpn/intro/)__
- __[EVPN Multihoming](https://learn.srlinux.dev/tutorials/evpn-mh/basics/)__
- __[L3 EVPN](https://learn.srlinux.dev/tutorials/l3evpn/rt5-only/overlay/)__


