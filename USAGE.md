# usage of katran library.
main class to interact with L4 forwarding plane is located in
lib/KatranLb.h. katran consists of two parts:
1. BPF code for actual forwarding (which is used in XDP attachment
  point; code located at lib/bpf/ folder)
2. cpp library to interact w/ BPF forwarding plane

# terminology
1. VIP - Virtual IP Address - IP address of the service. However, katran extends
this definition of VIP to also include port and
protocol (TCP or UDP), aside from just IP address
2. Real - IP address of backend server, where the traffic would be redirected

# example of usage.
All methods in KatranLb.h are well documented, but high level overview of
library usage could be describe in following steps.

1. initialize katran w/ config
2. load and attach bpf program
3. optionally (if enabled) add healthchecking endpoints
4. add VIP to forwarding plane w/ specified params
5. add reals to the VIP (either in batch or one by one. Reals could have
  different "weights" (to configure how much traffic))
6. if server goes out of service (e.g. due to maintenance) - it could be done either
by removing real from VIP or by setting up weight equal to zero for specified
real.


## initialize katran w/ config
katran's XDP forwarding plane can work in two modes:
1. "standalone" - when we attach katran directly to the interface
2. "shared" - this is a special mode that allows to run multiple XDP programs
from single interface. It is achieved by attaching special
("root", see example in lib/bpf/root_kern.h) XDP program; whole purpose of this
program is to try run another XDP programs from program array, and, as a last
step, if there is no registered bpf programs, send packet to the kernel

See EXAMPLE.md for example of usage in standalone and shared modes

katran config structure (from lib/KatranLbStruct.h)

```
struct KatranConfig {
  std::string mainInterface;
  std::string v4TunInterface;
  std::string v6TunInterface;
  std::string balancerProgPath;
  std::string healthcheckingProgPath;
  std::vector<uint8_t> defaultMac;
  uint32_t priority = kDefaultPriority;
  std::string rootMapPath = kNoExternalMap;
  uint32_t rootMapPos = kDefaultKatranPos;
  bool enableHc = true;
  uint32_t maxVips = kDefaultMaxVips;
  uint32_t maxReals = kDefaultMaxReals;
  uint32_t chRingSize = kLbDefaultChRingSize;
  bool testing = false;
  uint64_t LruSize = kDefaultLruSize;
  std::vector<int32_t> forwardingCores;
  std::vector<int32_t> numaNodes;
};
```
(additional description for the field is available in KatranLbStruct.h file)

1. mainInterface - name of the main interface; this is where XDP program is
attached, if katran works in "standalone" mode.
2. v4TunInterface and v6TunInterface - name of the tunneling interface, if
forwarding of healthchecks is enabled.
3. balancerProgPath - path to the object file which contains katran's bpf
forwarding plane
4. healthcheckingProgPath - path to the object file which contains katran's
bpf program for healthchecks forwarding
5. defaultMac - mac address of default router. katran "offloads" forwarding to
the top of the rack switch (by simply sending everything to it by default)
6. priority - TC's filter priority for healthchecking bpf program
7. rootMapPath - path to pinned program array of root XDP program (if katran
  is being used in "shared mode")
8. rootMapPos - position of katran in root program's program array
9. enableHc - flag to indicate if healthchecking forwarding plane should be
enabled or not.
10. maxVips - maximum number of VIPs supported by katran. It must be in sync
with configuration of forwarding plane (bpf program complie time constants.
see bpf specific configs bellow)
11. maxReals - maximum number of Real servers. It must be in sync w/ configuration
of forwarding plane.
12. chRingSize - size of consistent hashing ring size. It must be in sync w/
configuration of forwarding plane and, since it uses Maglev's hashing
algorithm it also must be a prime number.
13. testing - flag, which is indicates that this is test run or not. During a test-
run KatranLb library doesn't communicate with kernel through syscalls (and
therefore doesn't require root privileges. It is used only for unittesting)
14. LruSize - size of connection tracking table
15. forwardingCores - id of cpu cores which are responsible for the packet
forwarding. When you have multi queue NIC it could have less RX queues configured
for [RSS](https://github.com/torvalds/linux/blob/master/Documentation/networking/scaling.txt) than CPUs on the server.
in this case [best practice is to "pin"/map IRQs of the NIC to the certain CPUs.](https://github.com/torvalds/linux/blob/master/Documentation/IRQ-affinity.txt)
we store this mapping in this vector.
16. If server has multiple CPU sockets/NUMA domains - you can provide hints
of forwarding cores to NUMA node mappings

After populating this KatranConfig structure, next step is to
create an instance of KatranLb:
```
katran::KatranLb lb(config);
```

## load and attach bpf program

After you create an instance of KatranLb you need to load bpf program into the
kernel. At this step bpf in-kernel bpf verifier will run and report either
success or failure (if bpf program is "unsafe" from it's point of view;
library throws on failure)
```
lb.loadBpfProgs();
```

When program is successfully loaded - it can be attached (depending on the config,
either to interface directly (in "standalone" mode) or registered in program
array (in "shared" mode))

```
lb.attachBpfProgs();
```

## optionally (if going to be used) add healthchecking endpoints
katran's healtchecking forwarding plane, if enabled, works in such a way that all
packets with specified socket mark (which can be added with
setsockopt syscall (level SOL_SOCKET, optname SO_MARK)) will be forwarded to
configured real.
see EXAMPLE.md for more info on healthchecks.

socket mark to real server mapping can be added with this helpers
(in this example all packets with socket mark 100 would be forwarded to
server with ip address 10.0.0.1. with socket mark 200 - to fc00::1):
```
lb.addHealthcheckerDst(100, "10.0.0.1");
lb.addHealthcheckerDst(200, "fc00::1");
```
See lib/KatranLb.h for more info on how to delete this mapping or retrieve
currently configured ones.

## add VIP to forwarding plane w/ specified params
VIP is described by VipKey class (from lib/KatranLbStruct.h)
```
class VipKey {
 public:
  std::string address;
  uint16_t port;
  uint8_t proto;
 ...other methods...
}
```
To add a VIP you need to populate this class with intended values
(e.g. if you want to configure TCP VIP toward HTTP port this values are going to
be:)

```
katran::VipKey vip;
vip.address = "10.0.0.1";
vip.port = 80;
vip.proto = IPPROTO_TCP; // IPPROTO_TCP defined in linux/in.h
```
If you want to have a service, where all packets to specified IP address (but
different destination port) must be forwarded to the real servers - you need to
use port equal to 0 in VipKey. for example if you configure VipKey as:

```
vip.address = "10.0.0.1";
vip.port = 0;
vip.proto = IPPROTO_TCP;
```
then all packets with e.g. destination port 80, 443, 22 etc are going to be
forwarded to the real servers (by default if there is no match w/ port - packets
would be send to the kernel for further processing)


Next step is to add a VIP to load balancer

```
lb.addVip(vip);
```
The same function can be called w/ optional "flag" parameter.
‘flag’ can be used to specify some special conditions of this VIP. Currently the supported options are:

0 - default value. Default vip uses connection table and use
src port and src addresses of the packet for hashing
1 - HASH_NO_SRC_PORT, this flag removes src port from hashing calculation. This
allows packets from same source address, but different ports to end up on the
same destination server (some applications requires this: e.g. nfs or gfs)
2 - LRU_BYPASS - disable connection table lookup/update for this VIP
4 - QUIC_VIP - this is a VIP for QUIC protocol. Load balancing is going to be
done based on connection-id field. This is not fully stable/actively being developed codepath
(as IETF QUIC is still in developing stage and standard has not been finalized yet)
8 - HASH_DPORT_ONLY - use only destination port for hashing. In this case
only destination port is going to be used for hashing, so different clients
(different src address/src port) with the same destination port would end
up on the same real server (usually VOIP based protocols needs this)

See lib/KatranLb.h for more information (e.g. how to delete or modify VIP)

## add reals to the VIP

reals are described with NewReal struct (from lib/KatranLbStruct.h)
```
struct NewReal {
  std::string address;
  uint32_t weight;
};
```
for example, if you want to add a real with ip address 10.10.0.1 and weight 10
you need do something similar to:

```
katran::NewReal real;
real.address = "10.10.0.1";
real.weight = 10;
```
weight in this context is "amount of traffic to be sent to this
real". e.g. if server1 has 10x more weight than server2, it will receive 10x
more traffic. For best (most fair) load balancing sum of weights of all reals
for particular VIP should be equal to consistent hash ring size (by default
65537; controlled with chRingSize config param)

You can add real only for already existing VIP.
There are few ways to add real to the VIP. You can add reals one by one:
```
lb.addRealForVip(real, vip);
```
If you want to change weight of already added real, you just run this method
Again for that real. e.g.

```
real.weight = 100;
lb.addRealForVip(real, vip);
```

However, if you need to add (or delete) more than one real, it's better to use
modifyRealsForVip method, that allows to do this in a batch

```
lb.modifyRealsForVip(katran::ModifyAction::ADD, <vector of reals>, vip);
```

See lib/KatranLb.h for more information (e.g. how to get/delete reals for VIP)

## maintenance (or how to "drain" a real server)
To "drain" a server (so that new traffic won’t be sent there) you can either
remove real from the vip or (if you are expecting it to be back online soon)
you can change its weight to be 0.

```
real.weight = 0;
lb.addRealForVip(real, vip);
```

However, this drain will affect only new connections towards this real (unless
VIP was created with LRU_BYPASS flag), all established ones are still going to be
routed to this real (so that you can "drain" those connections, w/o affecting existing
sessions)

# Compile time bpf forwarding plane configurations.
katran contains two major part: userspace for all housekeeping, and bpf program
which is loaded into the kernel. Most of the bpf configuration must
be in sync w/ the same params in userspace (e.g. max reals, max vips,
consistent hash ring size). bpf related configurations are defined and described
in lib/bpf/balancer_consts.h. You can change them by providing -D flag during
bpf compilation time (e.g. by adding it in lib/Makefile-bpf)
