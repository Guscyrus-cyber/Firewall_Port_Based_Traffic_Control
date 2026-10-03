**Firewall Lab — Port-Based Firewall Traffic Control**

Allow, Deny, and Verify Services

Introduction

This lab demonstrates how a firewall controls network traffic based on TCP/UDP ports and firewall rules. Different services will be tested to determine whether traffic is allowed (ACCEPTED) or blocked (DROPPED/REJECTED).

The lab will simulate the kind of verification a SOC analyst or network security analyst performs when reviewing firewall policy, exposed services, and potentially dangerous ports.

Ports I will test

| Port       | Service | Typical Purpose                |
|------------|---------|--------------------------------|
| 22/TCP     | SSH     | Secure remote administration   |
| 23/TCP     | Telnet  | Insecure remote administration |
| 53/TCP/UDP | DNS     | Domain-name resolution         |
| 80/TCP     | HTTP    | Unencrypted web traffic        |
| 443/TCP    | HTTPS   | Encrypted web traffic          |
| 445/TCP    | SMB     | Windows file/printer sharing   |
| 3389/TCP   | RDP     | Windows Remote Desktop         |

A realistic policy could intentionally produce results such as:

| Port | Firewall Decision |
|------|-------------------|
| 22   | ACCEPT            |
| 23   | DROP              |
| 53   | ACCEPT            |
| 80   | ACCEPT            |
| 443  | ACCEPT            |
| 445  | DROP              |
| 3389 | DROP              |

I won't simply assume those results—I will create the firewall rules and prove them through testing.

Tools

The primary tools will be:

macOS Terminal — commands and traffic generation\
Docker — isolated Linux hosts/network environment\
iptables — firewall rules inside the Linux firewall container\
Netcat (nc) — testing whether particular TCP ports can be reached\
curl — HTTP/HTTPS testing\
dig/nslookup — DNS testing\
iptables counters/logs — evidence showing which packets matched firewall rules

Key terminology

Port — A logical number identifying a network service on a host.

Firewall Rule — A security instruction specifying what network traffic should be permitted or blocked.

ACCEPT — The firewall permits the packet to continue toward its destination.

DROP — The firewall silently discards the packet.

REJECT — The firewall blocks the packet but sends a response telling the sender that the connection was refused.

Inbound Traffic — Traffic entering a system or protected network.

Outbound Traffic — Traffic leaving a system or protected network.

TCP — Connection-oriented transport protocol used by services such as SSH, HTTP, HTTPS, SMB, and RDP.

UDP — Connectionless transport protocol commonly used by services such as DNS.

Listening Port — A port on which an application is actively waiting for incoming connections.

Attack Surface — The collection of services, ports, applications, and interfaces that could potentially be attacked.

Least Privilege / Least Exposure — Allowing only the ports and services actually required while blocking unnecessary access.

SOC Scenario

A security team has discovered that a newly deployed server has several network services reachable from another network.

The SOC must determine:

Which services are intentionally permitted, which are blocked by firewall policy, and whether any unnecessary high-risk administrative services are exposed.

After configuring and testing the firewall, there should be approximately 10–15 SOC analyst questions based on the evidence I generate.

Examples will include things such as identifying:

Which ports were accessible\
Which ports were blocked\
Why Telnet should normally be denied\
Why exposing SMB externally is dangerous\
Whether a failed connection indicates a firewall block or simply a service not listening\
Which firewall rule caused a packet to be dropped\
Whether SSH exposure is justified\
Which firewall rule should be investigated if RDP unexpectedly becomes reachable

Step 1 — Verify Docker is running

Because this lab will use isolated systems rather than modifying the firewall configuration of the MacBook itself, first I open Docker Desktop.

Then I open Terminal and I run: docker –version

and: docker ps

The output confirms:

Docker 29.6.1 is installed and running.\
docker ps works correctly.\
The containers from the previous Firewall-Protected DMZ Lab are still running.\
My Wazuh containers are also running.

I will not modify or reuse the previous DMZ containers. This new lab will get its own isolated Docker network and uniquely named containers, so the previous lab remains intact. (Image 1)\

Step 2 — Create the New Firewall Lab Network

In the terminal, I run:\
\
docker network create --subnet=172.30.0.0/24 port-firewall-net\
\
This creates a dedicated network:

Network: port-firewall-net\
Subnet: 172.30.0.0/24

Then I verify it: docker network inspect port-firewall-net

I am creating a small isolated network specifically for the new firewall experiment.

Later I will place the test systems inside it and deliberately configure firewall rules such as:

TCP 22 → ACCEPT

TCP 23 → DROP

UDP/TCP 53 → ACCEPT

TCP 80 → ACCEPT

TCP 443 → ACCEPT

TCP 445 → DROP

TCP 3389 → DROP

Then I’ll actually send traffic toward those ports and determine whether the firewall behaves according to policy.\
\
The output confirms:

Network: port-firewall-net \
Subnet: 172.30.0.0/24 \
Gateway: 172.30.0.1 \
Driver: bridge \
IPv4 enabled\
No lab containers attached yet ("Containers": {}) — exactly what I expect. (Image 2)

Step 3 — Create the Firewall Container

I create the Linux system that will act as our firewall.

I run this command:

docker run -dit \\

--name port-firewall \\

--network port-firewall-net \\

--ip 172.30.0.10 \\

--cap-add=NET_ADMIN \\

alpine sh

Then I verify it: docker ps --filter name=port-firewall
\
What this does

port-firewall = name of the firewall machine.

172.30.0.10 = its fixed IP address.

NET_ADMIN = gives the container permission to create and modify firewall/network rules.

alpine = lightweight Linux operating system used for the firewall.

The output confirms that the new container is running:

Container: port-firewall

Image: alpine

Status: Up

IP: 172.30.0.10

Network: port-firewall-net

This is now the first host attached to the network I created in Step 2. (Image 3)\

Step 4 — Install the Firewall and Testing Tools

The basic Alpine image does not include iptables by default. I install it inside the firewall container.

I run: docker exec port-firewall apk add --no-cache iptables iproute2 curl

Then I verify iptables: docker exec port-firewall iptables --version

Why iptables?

iptables is the tool I use to create rules such as:

Port 22 → ACCEPT

Port 23 → DROP

Port 80 → ACCEPT

Port 445 → DROP

Port 3389 → DROP

This is where the actual firewall configuration portion of the lab begins.\
\
The output confirms all three packages installed successfully, including:

iptables 1.8.13

iproute2

curl 8.22.0

And most importantly:

iptables v1.8.13 (nf_tables)

That means the Linux firewall container is ready to create firewall rules.\
(Image 4)\

Step 5 — Check the Firewall Before Adding Rules

I am going to inspect the firewall's current/default state. This gives me a baseline to compare against later.

I run: docker exec port-firewall iptables -L -n -v

### What this command means

iptables → firewall management tool\
-L → List the firewall rules\
-n → show numeric IP addresses/ports instead of resolving names\
-v → Verbose, showing additional information including packet and byte counters

The output shows:

INPUT → policy ACCEPT

FORWARD → policy ACCEPT

OUTPUT → policy ACCEPT

There are also no individual firewall rules yet underneath those chains. This means that, at this moment, iptables itself is not intentionally blocking any traffic. This baseline will be useful later because I'll see these empty chains become populated with specific ACCEPT and DROP rules.

ACCEPT does not automatically mean a port is open. A firewall may permit port 22, for example, but if no SSH service is listening on port 22, the connection still won't succeed. I'll deliberately create listening services so I can distinguish “firewall blocked it” from “nothing was listening.” (Image 5)


Step 6 — Create the Test Server

Now I create a second container that will eventually provide the services/ports that the firewall policy will test.

I run:

docker run -dit \\

--name port-test-server \\

--network port-firewall-net \\

--ip 172.30.0.20 \\

alpine sh

Then I verify both lab containers: docker ps --filter name=port-

The output confirms both containers are running:

port-firewall → Up 22 hours

port-test-server → Up

Current topology:

port-firewall-net (172.30.0.0/24)

│

├── port-firewall

│ 172.30.0.10

│

└── port-test-server

172.30.0.20 (Image 6)

Step 7 — Install Network Testing Tools on the Test Server


I need tools inside port-test-server so it can provide/listen on ports and later help to verify firewall behavior.

I run:

docker exec port-test-server apk add --no-cache \\

netcat-openbsd curl bind-tools iproute2

After installation, I verify Netcat:

docker exec port-test-server nc -h

### Why Netcat?

nc (Netcat) is particularly useful in this lab because I can make the test server listen on a specific TCP port.

For example, later I can create a listener on:

22 SSH

23 Telnet

80 HTTP

443 HTTPS

445 SMB

3389 RDP

That doesn't mean I am installing real SSH, Telnet, SMB, or RDP servers. Instead, Netcat lets me to create controlled listening ports so I can test whether the firewall permits or blocks traffic to those port numbers.

The installation finished successfully with: OK: 25.1 MiB in 50 packages

And the second output confirms OpenBSD Netcat 1.234-1 is installed and working. The long text I see after nc -his simply Netcat's help menu—not an error. (Image 7 and 8)\

Step 8 — Create the Listening Test Ports

Now I’ll make the test server listen on the selected TCP ports. This gives me known-open ports against which I can test the firewall.

I run:

docker exec port-test-server sh -c '

for p in 22 23 80 443 445 3389; do

nc -lk -p \$p \>/dev/null 2\>&1 &

done

'I’m intentionally leaving 53/DNS for a separate UDP/TCP test because DNS behaves differently from these TCP services.

Then I verify the listeners: docker exec port-test-server ss -lnt

The output confirms that all six TCP test ports are actively listening:

80 → LISTEN

22 → LISTEN

23 → LISTEN

445 → LISTEN

443 → LISTEN

3389 → LISTEN

0.0.0.0 means Netcat is listening on all IPv4 interfaces inside the test-server container. The 127.0.0.11:40437 entry is Docker's internal DNS-related service; it is not one of the test ports we created.

This is an important baseline: I now know these ports really have listeners. Therefore, later when I deliberately block a port, I can attribute the failure to the firewall policy rather than to the absence of a listener.\
(Image 9)

Step 9 — Create the Client Container


I now need a third machine that will generate the connection attempts.

I run:

docker run -dit \\

--name port-client \\

--network port-firewall-net \\

--ip 172.30.0.30 \\

alpine sh

Then I verify all three:

docker ps --filter name=port-

The lab will then contain:

port-firewall-net (172.30.0.0/24)

172.30.0.30

port-client

│

│ Test traffic

▼

172.30.0.10

port-firewall

│

▼

172.30.0.20

port-test-server

One important networking point: because all three containers currently share the same Docker bridge, simply connecting port-client directly to 172.30.0.20 would bypass port-firewall. Before I perform the actual ACCEPT/DROP tests, I'll configure the topology/routing so the firewall is genuinely in the traffic path. Otherwise, the lab could appear to demonstrate firewall filtering when it actually doesn't.

The output confirms all three lab containers are running:

port-client → Up

port-test-server → Up

port-firewall → Up

Before I start testing ports, I need to correct an important topology issue: right now all three are on the same subnet, so traffic from port-client to port-test-server can travel directly and would not pass through port-firewall. (Image 10)\

Step 10 — Install Testing Tools on the Client


First, I prepare port-client for connectivity testing.

I run:

docker exec port-client apk add --no-cache netcat-openbsd iproute2

Then I verify Netcat:

docker exec port-client nc -h

After this, I'll restructure the network so the path actually becomes:

CLIENT

│

▼

FIREWALL

│

▼

SERVER

That is essential because when I later see something like 22 = ACCEPT and 445 = DROP, I'll know the result genuinely came from the firewall.

The first output confirms netcat-openbsd and iproute2 installed successfully, and the second confirms Netcat is working on port-client.

I now have all three systems prepared:

port-client 172.30.0.30 → generates test traffic

port-firewall 172.30.0.10 → applies firewall rules

port-test-server 172.30.0.20 → provides test ports (Image 11 and 12)

Step 11 — Create Two Separate Network Segments


This is an important step. Right now, client and server share the same network, so they can communicate without passing through the firewall.

I 'll create a second Docker network so the firewall can sit between two network segments.

I run:

docker network create --subnet=172.31.0.0/24 port-server-net

Then I verify it:

docker network inspect port-server-net

I’m preparing this architecture:

CLIENT SIDE SERVER SIDE

172.30.0.0/24 172.31.0.0/24

port-client

172.30.0.30

│

│

▼

port-firewall

172.30.0.10

│

│ Firewall will have

│ a second interface

▼

172.31.0.10

│

▼

port-test-server

172.31.0.20

The firewall will therefore become the bridge/router between the two security zones. Traffic will have to cross it, allowing iptables to make the ACCEPT/DROP decision.

The output confirms:

Network: port-server-net

Subnet: 172.31.0.0/24

Gateway: 172.31.0.1

Driver: bridge

IPv4: enabled

Containers: none yet

The "Containers": {} is expected again because I have created the server-side network, but haven't moved/connected the server and firewall to it yet. (Image 13)

Top of Form

Bottom of Form

Top of Form

Step 12 — Connect the Firewall to the Server Network

The firewall needs two network interfaces. one facing the client and one facing the server.

It already has:

Client side:

port-firewall-net

172.30.0.10

Now give it its second interface:

docker network connect \\

--ip 172.31.0.10 \\

port-server-net \\

port-firewall

Then I verify the firewall's addresses:

docker exec port-firewall ip -4 addr

### What I am building

After this step, the firewall will effectively have two doors:

CLIENT NETWORK SERVER NETWORK

172.30.0.0/24 172.31.0.0/24

│ │

│ 172.30.0.10 172.31.0.10 │

└────────── PORT-FIREWALL ──────────┘

This is fundamental to how a real network firewall/router works: one interface receives traffic from one network, and another interface sends permitted traffic toward the other network.

The firewall now has two active IPv4 interfaces:

eth0 → 172.30.0.10/24 client side

eth1 → 172.31.0.10/24 server side

The firewall is now connected to both network segments. (Image 14)

Bottom of Form

Step 13 — Move the Test Server to the Server-Side Network

Right now port-test-server is still attached to the original client-side network, so I need to move it behind the firewall.

First I disconnect it from the old network:

docker network disconnect port-firewall-net port-test-server

Then I connect it to the new server-side network with its new IP:

docker network connect \\

--ip 172.31.0.20 \\

port-server-net \\

port-test-server

Then I verify its IPv4 address:

docker exec port-test-server ip -4 addr

The outputs confirm that port-test-server has been successfully removed from the original 172.30.0.0/24 network and now has:

eth0 → 172.31.0.20/24

So I now have the proper two-network architecture:

CLIENT NETWORK SERVER NETWORK

172.30.0.0/24 172.31.0.0/24

port-client

172.30.0.30

│

▼

172.30.0.10

┌─────────────────┐

│ port-firewall │

└─────────────────┘

172.31.0.10

│

▼

port-test-server

172.31.0.20 (Image 15)


Step 14 — Enable IP Forwarding on the Firewall

The firewall has interfaces on both networks, but Linux must also be told that it is allowed to forward packets between those networks.

First I check the current setting:

docker exec port-firewall sysctl net.ipv4.ip_forward

It will report either:

net.ipv4.ip_forward = 0

or:

net.ipv4.ip_forward = 1\
\
The result shows:

net.ipv4.ip_forward = 1

That means IP forwarding is already enabled inside port-firewall. I do not need to change it.

In simple terms, the firewall is capable of receiving a packet on its client-side interface and forwarding it out through its server-side interface.\
(Image 16)\

Step 15 — Add Routes Through the Firewall


Now I need to tell the client and server that traffic destined for the other subnet should go through port-firewall.

On the client, I add a route to the server network:

docker exec port-client ip route add 172.31.0.0/24 via 172.30.0.10

On the server, I add the return route to the client network:

docker exec port-test-server ip route add 172.30.0.0/24 via 172.31.0.10

Then I display both routing tables:

docker exec port-client ip route

docker exec port-test-server ip route

So, establishing this path:

port-client

172.30.0.30

│

│ route 172.31.0.0/24

▼

172.30.0.10

┌─────────────────┐

│ port-firewall │

└─────────────────┘

172.31.0.10

│

│ route 172.30.0.0/24

▼

port-test-server

172.31.0.20

This is the crucial routing step that forces the cross-subnet traffic through the firewall.

The outputs show exactly what happened. Step 15 is not complete yet, but nothing is broken.

Both route commands failed with:

RTNETLINK answers: Operation not permitted

That happened because port-client and port-test-server were created without the Linux NET_ADMIN capability. Only port-firewall was given --cap-add=NET_ADMIN.

The routing tables confirm that no new routes were added:

port-client

default via 172.30.0.1

172.30.0.0/24

port-test-server

default via 172.31.0.1

172.31.0.0/24 (Image 17)

Step 15A — Add the Routes with Privileged Exec

I don’t need to delete or rebuild the containers. First I try running the route commands with temporary elevated privileges.

I run:

docker exec --privileged port-client \\

ip route add 172.31.0.0/24 via 172.30.0.10

Then I run:

docker exec --privileged port-test-server \\

ip route add 172.30.0.0/24 via 172.31.0.10

Now I verify:

docker exec port-client ip route

And I run:

docker exec port-test-server ip route

I want the client to show something similar to:

172.31.0.0/24 via 172.30.0.10

and the server:

172.30.0.0/24 via 172.31.0.10

So the failure I saw is actually useful lab evidence: Linux prevents an ordinary container from changing its routing table unless it has sufficient network-administration privileges.

The important lines in the outputs are:

port-client:

172.31.0.0/24 via 172.30.0.10 dev eth0

and:

port-test-server:

172.30.0.0/24 via 172.31.0.10 dev eth0

So traffic between the two lab networks is now explicitly routed through port-firewall. (Image 18)
\

Step 16 — Test Baseline Connectivity Through the Firewall

Before creating any ACCEPT/DROP rules, I prove that the client can reach the server through the firewall.

From port-client, I run:

docker exec port-client ping -c 4 172.31.0.20

Then I test one of the known listening ports—port 22:

docker exec port-client nc -zvw 2 172.31.0.20 22

### What I am testing

The path should be:

port-client

172.30.0.30

│

▼

port-firewall

172.30.0.10 → 172.31.0.10

│

▼

port-test-server

172.31.0.20:22

Because the firewall's FORWARD policy is still ACCEPT, I expect the ping to succeed and Netcat to report that TCP port 22 succeeded/open.

This establishes the pre-firewall-policy baseline. Later I deliberately allow some ports and drop others and compare the results.

The output gives me two important pieces of baseline evidence.

The ping test shows:

4 packets transmitted

4 packets received

0% packet loss

So the client can successfully reach the server across the firewall.

The TCP test shows:

Connection to 172.31.0.20 22 port \[tcp/ssh\] succeeded!

So TCP port 22 is reachable before I apply restrictive firewall rules. (Image 19)

Step 17 — Verify the Traffic Reached the Firewall


Before adding the rules, I inspect the firewall's FORWARD chain counters. This will provide evidence that traffic is actually passing through port-firewall.

I run:

docker exec port-firewall iptables -L FORWARD -n -v

I expect something similar to:

Chain FORWARD (policy ACCEPT ...)

but unlike the earlier baseline, the packet and byte counters should now be greater than zero because the ping and TCP connection crossed the firewall.

This is an important checkpoint: it proves I am not merely testing two containers that happen to communicate directly.

This result reveals something important. Step 17 did not give the evidence I expected.

The output says:

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)

So the port-firewall container's iptables FORWARD chain did not count the ping or TCP traffic from Step 16.

That means I should not proceed with ACCEPT/DROP rules yet. I need to verify the actual packet path first. Docker networking can sometimes route traffic through the host's virtual networking in a way that differs from the simple topology diagram. (Image 20)

Step 17A — Verify the actual route

I run this from the client:

docker exec port-client ip route get 172.31.0.20

I want to see something like:

172.31.0.20 via 172.30.0.10 dev eth0

Then I run:

docker exec port-test-server ip route get 172.30.0.30

I want the return path to show:

172.30.0.30 via 172.31.0.10 dev eth0

These commands tell me which next-hop Linux is actually selecting, rather than assuming it from the routing table.

Both routing decisions explicitly point to the firewall:

CLIENT → SERVER

172.31.0.20 via 172.30.0.10

↑

firewall

and the return path:

SERVER → CLIENT

172.30.0.30 via 172.31.0.10

↑

firewall

So the routes themselves are correct.

The strange part is that iptables showed 0 packets in the FORWARD chain despite successful communication. Rather than guessing, I will verify packet traversal directly. (Image 21)

