# SRL Local and Remote Mirroring
<div align="center" markdown>
<img src=https://github.com/user-attachments/assets/e54b2b85-da8e-46c7-8614-6eb04947b532 style="width:50%" />

<br><br>

<a href="https://codespaces.new/toweber/srl-mirroring?quickstart=1">
<img src="https://gitlab.com/rdodin/pics/-/wikis/uploads/d78a6f9f6869b3ac3c286928dd52fa08/run_in_codespaces-v1.svg?sanitize=true" style="width:50%"/></a>

**[Run](https://codespaces.new/toweber/srl-mirroring?quickstart=1) this lab in GitHub Codespaces for free**.  
[Learn more](https://containerlab.dev/manual/codespaces) about Containerlab for Codespaces.  
<small>Machine type: 2 vCPU · 8 GB RAM</small>
</div>

## Accessing the network elements

Once the lab has been deployed, the different SR Linux nodes can be accessed via SSH through their management IP address, given in the summary displayed after the execution of the deploy command. It is also possible to reach those nodes directly via their hostname, defined in the topology file. Linux clients cannot be reached via SSH, as it is not enabled, but it is possible to connect to them with a docker exec command.

```bash
# reach a SR Linux leaf or a spine via SSH
ssh admin@leaf1
ssh admin@spine1

# reach a Linux client via Docker
docker exec -it client1 bash
```

## Mirroring config
### Local mirror destination on Leaf1
```
set / interface ethernet-1/10
set / interface ethernet-1/10 admin-state enable
set / interface ethernet-1/10 subinterface 0
set / interface ethernet-1/10 subinterface 0 admin-state enable
set / interface ethernet-1/10 subinterface 0 type local-mirror-dest
set / interface ethernet-1/10 subinterface 0 local-mirror-destination
set / interface ethernet-1/10 subinterface 0 local-mirror-destination admin-state enable

set / system mirroring
set / system mirroring mirroring-instance 1
set / system mirroring mirroring-instance 1 admin-state enable
set / system mirroring mirroring-instance 1 mirror-source
set / system mirroring mirroring-instance 1 mirror-source interface ethernet-1/1
set / system mirroring mirroring-instance 1 mirror-source interface ethernet-1/1 direction ingress-egress
set / system mirroring mirroring-instance 1 mirror-destination
set / system mirroring mirroring-instance 1 mirror-destination local ethernet-1/10.0
```

### Remote mirror destination on Leaf2
```
set / acl acl-filter mirror-acl type ipv4
set / acl acl-filter mirror-acl type ipv4 entry 10
set / acl acl-filter mirror-acl type ipv4 entry 10 description "Match ICMP"
set / acl acl-filter mirror-acl type ipv4 entry 10 action
set / acl acl-filter mirror-acl type ipv4 entry 10 action accept
set / acl interface ethternet-1/1.0
set / acl interface ethternet-1/1.0 interface-ref
set / acl interface ethternet-1/1.0 interface-ref interface ethernet-1/1
set / acl interface ethternet-1/1.0 interface-ref subinterface 0
set / acl interface ethternet-1/1.0 input
set / acl interface ethternet-1/1.0 input acl-filter mirror-acl type ipv4
set / system mirroring
set / system mirroring mirroring-instance 1
set / system mirroring mirroring-instance 1 admin-state enable
set / system mirroring mirroring-instance 1 mirror-source
set / system mirroring mirroring-instance 1 mirror-source acl
set / system mirroring mirroring-instance 1 mirror-source acl acl-filter mirror-acl type ipv4
set / system mirroring mirroring-instance 1 mirror-source acl acl-filter mirror-acl type ipv4 entry 10
set / system mirroring mirroring-instance 1 mirror-destination
set / system mirroring mirroring-instance 1 mirror-destination remote
set / system mirroring mirroring-instance 1 mirror-destination remote encap l2ogre
set / system mirroring mirroring-instance 1 mirror-destination remote network-instance default
set / system mirroring mirroring-instance 1 mirror-destination remote tunnel-end-points
set / system mirroring mirroring-instance 1 mirror-destination remote tunnel-end-points source-address 10.0.1.2
set / system mirroring mirroring-instance 1 mirror-destination remote tunnel-end-points destination-address 192.168.1.10
```

## Verification
### Ping on Client1
```
client1# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.714 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.668 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.709 ms
```

### Mirror1
```
mirror1# tcpdump -nni eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
06:39:44.621663 IP 10.0.1.2 > 192.168.1.10: GREv0, length 102: IP 172.17.0.2 > 172.17.0.1: ICMP echo reply, id 34, seq 1, length 64
06:39:45.645693 IP 10.0.1.2 > 192.168.1.10: GREv0, length 102: IP 172.17.0.2 > 172.17.0.1: ICMP echo reply, id 34, seq 2, length 64
06:39:46.668930 IP 10.0.1.2 > 192.168.1.10: GREv0, length 102: IP 172.17.0.2 > 172.17.0.1: ICMP echo reply, id 34, seq 3, length 64
```

### Mirror2
```
mirror2# tcpdump -nni eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
06:39:44.621314 IP 172.17.0.1 > 172.17.0.2: ICMP echo request, id 34, seq 1, length 64
06:39:44.621808 IP 172.17.0.2 > 172.17.0.1: ICMP echo reply, id 34, seq 1, length 64
06:39:45.645338 IP 172.17.0.1 > 172.17.0.2: ICMP echo request, id 34, seq 2, length 64
06:39:45.645893 IP 172.17.0.2 > 172.17.0.1: ICMP echo reply, id 34, seq 2, length 64
06:39:46.668389 IP 172.17.0.1 > 172.17.0.2: ICMP echo request, id 34, seq 3, length 64
06:39:46.669156 IP 172.17.0.2 > 172.17.0.1: ICMP echo reply, id 34, seq 3, length 64

```
