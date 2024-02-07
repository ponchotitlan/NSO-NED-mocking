# NSO Netsim command mocking
## What's the deal with our netsims?

While learning or working with NSO (Network Services Orchestrator), you must have come across the concept of ```netsims```. These are very basic abstractions of data models which serve the purpose of dummy nodes for testing our NSO services.

In the gut, a netsim is just a small SSH server with an interactive command interface that mimicks a real device. Depending on the NED (Network Element Driver) used for its creation, a netsim will have different parameters that NSO can configure. These are all aligned to the specific data models (YANG) of that NED.

### Pros about netsims

1. Already available with any NSO installation
2. Easy and fast to create, manage and destroy
3. Onboarding on NSO is pretty straightforward

### Cons about netsims

1. It's all an illusion! No internal data/control plane
2. Won't accurately reflect cascading effects of your configs
3. Wion't change their status based on the configs given (Ex. setup OSPF protocol, port-status retrieval, etc)
3. Behavior is restricted to the rules within the data models of the NED they're based on

### So why do I want to use netsims in the end?

There are other options than having a real devices lab network for testing and pipeline purposes. Platforms such as [Cisco Modeling Labs (CMS)](https://www.cisco.com/c/en/us/products/cloud-systems-management/modeling-labs/index.html) serve this purpose. However, a better question than "why" go for netsims would be "when" to go for netsims. Given how easy it is to handle them, they are the go-to tooling when testing any of the following scenarios:

- **rfs configs validation:** Is the service package that I just coded really pushing the configs that I intend? Specially when it comes to ```rfs``` services, which face directly to the southbound levels.

- **Service compatibility:** Will the service that I just developed work with this specific version of the NED? After all, the NED will be the interface between NSO and our real network devices. In case there is a scenario that is not supported (ex. specific config, corner case, etc), a netsim should give us a clue so we can proactively take action.

- **Template generation:** It is very common to push configurations in a netsim device and then retrieve them from NSO's CDB (the built-in config state database) so that we know their structure and use it for our ```rfs``` services development later on.

> The ```ncs-netsim``` utility is the responsible of the netsim management. The 101 of this tooling is not contemplated in the scope of this document. You can find more information about it [in this link](https://developer.cisco.com/learning/labs/learn-nso-with-netsim/introduction/). 

## Leveling netsims up a little bit
### Live-status commands mockup

This is the situation: My service package needs to run some pre-checks in my interfaces to determine if there if there is already an OSPF link established.

Basically, we need to run this command and analyse the result:

```
Device# show ip ospf interface Ethernet 1/1
```

If there is an active OSPF established, we would expect the following output:

```
Ethernet1/1 is up, line protocol is up 
  Internet Address 10.3.252.113/30, Area O
  Label stack Primary Tabel 1 Backup Tabel 3 SRTE Tabel 10
  Process ID 100, Router ID 10.44.17.35, Network Type POINT_TO_POINT, Cost: 10
  LDP Sync Enabled, Sync Status: Achieved
  Transmit Delay is 1 sec, State POINT_TO_POINT, MTU 9100, MaxPktsz 1500
  Forward reference No, Unnumbered no, Bandwidth 100000000
  BFD enabled, BFD interval 50 msec, BFD multiplier 3, Mode: Default
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  Non-Stop Forwarding (NSF) enabled
    Hello due in 00:00:05:460
  Index 82/82, flood queue lenght 0
  Next 0(0)/0(0)
  Last flood scan length is 1, maximum is 48
  Last flood scan time is 0 msec, maximum is 14 msec
  LS Ack List: current length 0, high water mark 686
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 10.65.0.93
  Suppress hello for 0 neighbor(s)
  Message digest authentication enabled
    Youngest key id is 1
  Multi-area interface Count is 0
```

Now, what happens if we query this interface status via NSO from one of our netsims after pushing our OSPF configurations? First of all, NSO allows us to send raw CLI commands to our devices using the keyword ```live-status```:

```
admin@ncs# devices device <name> live-status exec any <internal command>
```

Nevertheless, we would get something like this from our netsim:

```
admin@ncs# devices device T1-IOS-ACC-C3K6-U-IOS-1 live-status exec any show ip ospf interface Ethernet 1/1
result 
------------------------------^
syntax error: missing display group
T1-IOS-ACC-C3K6-U-IOS-1# 
admin@ncs# 
```

As expected, our netsim doesn't know what to do. This is when we can enable the mocking of this behavior with a couple of simple steps.

First of all, navigate to the source location of your uncompressed NED folder and search for the directory ```netsim/```. In our case, we will be using the ```cisco-ios-cli-6.90``` NED:

```
/opt/ncs/packages/cisco-ios-cli-6.90/netsim
```

Now, add a ```.sh``` file with the following structure:

```
#!/bin/bash

cat <<EOF
Ethernet1/1 is up, line protocol is up 
  Internet Address 10.3.252.113/30, Area O
  Label stack Primary Tabel 1 Backup Tabel 3 SRTE Tabel 10
  Process ID 100, Router ID 10.44.17.35, Network Type POINT_TO_POINT, Cost: 10
  LDP Sync Enabled, Sync Status: Achieved
  Transmit Delay is 1 sec, State POINT_TO_POINT, MTU 9100, MaxPktsz 1500
  Forward reference No, Unnumbered no, Bandwidth 100000000
  BFD enabled, BFD interval 50 msec, BFD multiplier 3, Mode: Default
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  Non-Stop Forwarding (NSF) enabled
    Hello due in 00:00:05:460
  Index 82/82, flood queue lenght 0
  Next 0(0)/0(0)
  Last flood scan length is 1, maximum is 48
  Last flood scan time is 0 msec, maximum is 14 msec
  LS Ack List: current length 0, high water mark 686
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 10.65.0.93
  Suppress hello for 0 neighbor(s)
  Message digest authentication enabled
    Youngest key id is 1
  Multi-area interface Count is 0
EOF
```

Save this file in the same directory with any given name. In our case, we will name it as ```show_ip_ospf.sh```. Basically, this is a bash script which will only return in the standard stdout (a fancy way of saying "print in the console") whatever text we put in between these lines:

```
#!/bin/bash

cat <<EOF
👉🏽 Any given mockup text here! 👈🏽
EOF
```

Next, we need to let our netsims derived from this NED that whenever the command for interface OSPF is received, our small .sh script needs to be executed so that the mockup text is displayed.

For this purpose, open the file ```confd.c.cli``` and add the following lines to the xml tree:

```
<cmd name="interfacesOSPF" mount="show">
    <info>Show IP Interfaces OSPF status</info>
    <help>Show IP Interfaces OSPF status</help>
    <params>
      <any>
        <info>show ip ospf interface</info>
        <help>show ip ospf interface</help>
      </any>
    </params>
    <callback>
      <exec>
        <osCommand>./show_ip_ospf.sh</osCommand>
        <options>
          <noInput/>
          <pty>false</pty>
        </options>
      </exec>
    </callback>
  </cmd>
```

Next, navigate to the ```src/``` directory of your NED and compile it:

```
[root@d6615270f222 /]# cd /var/opt/ncs/packages/cisco-ios-cli-6.90/src/
[root@d6615270f222 src]# make clean all
```

Now, whenever you create a new netsim device and onboard it on NSO, the ```live-status``` command will provide the desired output. But what about the netsim devices that I already have? You can provide this functionality by copying the following files from your NED directory into their ```netsim/``` directory:

```
[root@d6615270f222 netsim]# cp confd.c.ccl /var/opt/ncs/netsim/T1-IOS-ACC-C3K6-U-IOS-1/T1-IOS-ACC-C3K6-U-IOS-1/
cp: overwrite ‘/var/opt/ncs/netsim/T1-IOS-ACC-C3K6-U-IOS-1/T1-IOS-ACC-C3K6-U-IOS-1/confd.c.ccl’? yes

[root@d6615270f222 netsim]# cp confd.i.ccl /var/opt/ncs/netsim/T1-IOS-ACC-C3K6-U-IOS-1/T1-IOS-ACC-C3K6-U-IOS-1/
cp: overwrite ‘/var/opt/ncs/netsim/T1-IOS-ACC-C3K6-U-IOS-1/T1-IOS-ACC-C3K6-U-IOS-1/confd.i.ccl’? yes
```

After doing this, the netsim device will provide the desired output when using the ```live-status``` command:

```
admin@ncs# devices device T1-IOS-ACC-C3K6-U-IOS-1 live-status exec any show ip ospf interface Ethernet 1/1

#!/bin/bash

cat <<EOF
Ethernet1/1 is up, line protocol is up 
  Internet Address 10.3.252.113/30, Area O
  Label stack Primary Tabel 1 Backup Tabel 3 SRTE Tabel 10
  Process ID 100, Router ID 10.44.17.35, Network Type POINT_TO_POINT, Cost: 10
  LDP Sync Enabled, Sync Status: Achieved
  Transmit Delay is 1 sec, State POINT_TO_POINT, MTU 9100, MaxPktsz 1500
  Forward reference No, Unnumbered no, Bandwidth 100000000
  BFD enabled, BFD interval 50 msec, BFD multiplier 3, Mode: Default
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  Non-Stop Forwarding (NSF) enabled
    Hello due in 00:00:05:460
  Index 82/82, flood queue lenght 0
  Next 0(0)/0(0)
  Last flood scan length is 1, maximum is 48
  Last flood scan time is 0 msec, maximum is 14 msec
  LS Ack List: current length 0, high water mark 686
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 10.65.0.93
  Suppress hello for 0 neighbor(s)
  Message digest authentication enabled
    Youngest key id is 1
  Multi-area interface Count is 0
EOF
```