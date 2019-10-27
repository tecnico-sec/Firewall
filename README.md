Instituto Superior Técnico, Universidade de Lisboa

**Network and Computer Security**

# Lab guide: Firewalls

## 0. Goals

- Configure a firewall using iptables

## 1. Introduction

Table 1 below shows the network topology configuration for this laboratory assignment. Based on the previous laboratory assignments of Virtual Networking and Traffic Analysis, _Initial configuration_ below on the left, the goal is to perform the necessary configuration changes to obtain the _Target configuration_ on the right.

For that, you should proceed as follows:

- Add a new Adapter 3 (enp0s9) to VM2 and attach it to a new Internal Network _sw-3_ (or change it if you already had a 3rd adapter on VM2).
- Attach Adapter 3 to the subnet 192.168.2.0/24 and set VM2's IP address as 192.168.2.254 on that adapter's configuration.
- Attach VM4's Adapter 1 to _sw-3_.
- Attach Adapter 1 to the subnet 192.168.2.0/24 and set VM4's IP address as 192.168.2.4 on that adapter's configuration. Do not forget to change the default gateway to be 192.168.2.254.

You should apdapt to your adapter names accordingly.

Please revise the previous lab assignments for instructions on how to obtain the initial configuration (left box of the table), taking into account whether you are using rnl-virt or VirtualBox.

| # Interface | Subnet | Adapter | | # Interface | Subnet | Adapter |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| __VM1__ |||||||
| 1 | 192.168.0.100 | enp0s3 || 1 | 192.168.0.100 | enp0s3
| __VM2__ |
| 1 | 192.168.0.10 | enp0s3 || 1 | 192.168.0.10 | enp0s3
| 2 | 192.168.1.254 | enp0s8 || 2 | 192.168.1.254 | enp0s8
| 3 | INTERNET | enp0s9 || 3 | __192.168.2.254__ | enp0s9
| __VM3__: |
| 1 | 192.168.1.1 | enp0s3 || 1 | 192.168.1.1 | enp0s3
| __VM4__: |
| 1 | 192.168.1.4 | enp0s3 || 1 | __192.168.2.4__ | enp0s3

_Table 1: Initial Configuration (from Virtual Networking and Traffic Analysis lab) on the left, and Target Configuration for this firewall lab on the right._

## 2. iptables

The native firewall software in Linux is part of the kernel. However, you can use the iptables tool (man iptables) to manage its rules. All the rules below should be applied in VM2 unless it is said otherwise.

Start by flushing all existing rules (if there are any):

```bash
$ sudo /usr/sbin/iptables –F
```

### 2.1. Simple Rules

Experiment with some simple rules in VM2.

#### 2.1.1. Reject ICMP packets

The following command adds a rule to drop all incoming ICMP packets.

```bash
$ sudo /usr/sbin/iptables –A INPUT –p icmp –j DROP
```

This new rule can be seen by listing all rules managed by iptables:

```bash
$ sudo /usr/sbin/iptables –L
```

Test this new rule by sending a ping from VM3 to VM2.

- Were you able to see (on VM3) the ping being performed to VM2?
- Were you able to see (on VM2) the ping from VM3? Why not?
- Can you ping VM3 from VM4?
- And VM4 from VM3?

Use one of the following commands to erase this rule from VM2:

```bash
$ sudo /usr/sbin/iptables –D INPUT 1
$ sudo /usr/sbin/iptables –D INPUT –p icmp –j DROP
```

#### 2.1.2. Ignore telnet connections

Confirm that you can establish a telnet connection to VM2 (for example, try from VM1). Block these connections using the following command (in VM2).

```bash
$ sudo /usr/sbin/iptables –A INPUT –p tcp –-dport 23 –j DROP
```

Check whether telnet connections to VM2 are still possible.

Delete the previous rule by executing one of the following commands:

```bash
$ sudo /usr/sbin/iptables –D INPUT 1
$ sudo /usr/sbin/iptables –D INPUT –p tcp –-dport 23 –j DROP
```

#### 2.1.3. Ignore telnet connections from specific IP addresses

Ignore telnet connections from VM1:

```bash
$ sudo /usr/sbin/iptables –A INPUT –p tcp –s [host address] –-dport 23 –j DROP
```

Check that all machines except VM1 are able to open a telnet connection with VM2.

#### 2.1.4. Ignore telnet connections from a specific subnet

Ignore telnet connections from the subnet that includes VM4.

```bash
$ /usr/sbin/iptables –A INPUT –p tcp –s 192.168.2.0/24 –-dport 23 –j DROP
```

At this point you should not be able to open a telnet connection to VM2 from VM4.

Delete all existing rules.

```bash
$ sudo /usr/sbin/iptables –F
```

### 2.2 Redirect connections

The previous exercises used the INPUT chain from the Filter table. This chain affects the packets addressed to the machine where the rule is being defined.

We will now use the PREROUTING chain in the NAT table in order to redirect network packets (and perfrom DNAT and SNAT translations). To list all the rules of the NAT table use:

```bash
$ sudo /usr/sbin/iptables -t nat -L
```

Run

```bash
$ sudo /usr/sbin/iptables -t nat -A PREROUTING -–dst 192.168.0.10 -p tcp --dport 23 –j DNAT  --to-destination 192.168.1.1
```

Make a telnet connection from VM1 to VM2.

- Are you in VM2? Run `netstat –t` command on VM2.
- Where are you then?
- Confirm that the connection was established between VM1 and VM3 using the `netstat –t` command on VM3.

In order to redirect http traffic to VM3 change from port 23 to 80 on the previous iptables command.

Use a browser in VM1 and go to `http://192.168.0.10` (this is VM2's address).

- Run `netstat –t` onm VM3 to confirm that the connection is in fact between VM1 and VM3:

Delete now all existing rules:

```bash
$ sudo /usr/sbin/iptables –F
$ sudo /usr/sbin/iptables -t nat –F
```

## 3. iptables: Internal Network + DMZ

Use iptables to configure the following requirements:

- VM1 is an external machine:
  - VM1 will only be able to open ssh connections(port 22) and http connections (port 80) with VM2.
- VM2 is the firewall
  - All http connections (port 80) are redirected to VM3.
  - All ssh connections from the external network are redirected to VM4.
  - Requests from the internal network 192.168.2.0/24 are only accepted if destined to the ssh port.
  - All other traffic is rejected.
- VM3 is a Web server in a DMZ:
  - Accepts http connections from both the internal and external networks.
  - Accepts ssh connections from the internal network.
  - Does not start any new connections.
- VM4 is an internal machine:
  - Accepts ssh requests.
  - Is able to open ssh connections to both external network and DMZ.

**Acknowledgments**

Adapted by: Pedro Adão

----

[SIRS Faculty](mailto:meic-sirs@disciplinas.tecnico.ulisboa.pt)
