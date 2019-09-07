## cumulus-segment-routing

### Summary:

  - Cumulus Linux 3.7.8
  - Underlying Topology Converter to 4.7.0
  - Tested against Vagrant 2.1.5 on Mac and Linux. Windows is not supported.
  - Tested against Virtualbox 5.2.32 on Mac 10.14
  - Tested against Libvirt 1.3.1 and Ubuntu 16.04 LTS

### Description:

This is an Ansible demo of the following Cumulus Linux Segment Routing example:

https://docs.cumulusnetworks.com/display/DOCS/Segment+Routing#SegmentRouting-ExampleConfiguration

### Install and Setup Virtualbox on Mac

Setup Vagrant for the first time on Mojave, MacOS 10.14.6

1. Install Homebrew 2.1.9 (This will also install Xcode Command Line Tools)

    https://brew.sh

2. Install Virtualbox (Tested with 5.2.32)

    https://www.virtualbox.org

I had to go through the install process twice to load the proper security extensions (System Preferences > Security & Privacy > General Tab > "Allow" on bottom)

3. Install Vagrant (Tested with 2.1.5)

    https://www.vagrantup.com

### Install and Setup Linux / libvirt demo environment:

First, make sure that the following is currently running on your machine:

1. This demo was tested on a Ubuntu 16.04 VM w/ 4 processors and 32Gb of Diagram

2. Following the instructions at the following link:

    https://docs.cumulusnetworks.com/cumulus-vx/Development-Environments/Vagrant-and-Libvirt-with-KVM-or-QEMU/

3. Download the latest Vagrant, 2.1.5, from the following location:

    https://www.vagrantup.com/

### Initializing the demo environment:

1. Copy the Git repo to your local machine:

    ```git clone https://github.com/chronot1995/cumulus-segment-routing```

2. Change directories to the following

    ```cumulus-segment-routing```

3a. Run the following for Virtualbox:

    ```./start-vagrant-vbox-poc.sh```

3b. Run the following for Libvirt:

    ```./start-vagrant-libvirt-poc.sh```

### Running the Ansible Playbook

1a. SSH into the Virtualbox oob-mgmt-server:

    ```cd vx-vbox-simulation```   
    ```vagrant ssh oob-mgmt-server```

1a. SSH into the Libvirt oob-mgmt-server:

    ```cd vx-libvirt-simulation```   
    ```vagrant ssh oob-mgmt-server```

2. Copy the Git repo unto the oob-mgmt-server:

    ```git clone https://github.com/chronot1995/cumulus-segment-routing```

3. Change directories to the following

    ```cumulus-segment-routing/automation```

4. Run the following:

    ```./provision.sh```

This will run the automation script and configure the environment.

### Troubleshooting

Helpful FRR and MPLS troubleshooting commands:

- show ip bgp summary
- show ip route
- show mpls status
- show mpls table
- show mpls fec

Helpful Linux troubleshooting commands:

- ip r s

### Segment Routing Troubleshooting

1. "show ip bgp summary"

```IPv4 Labeled Unicast Summary:
BGP router identifier 10.1.1.1, local AS number 65111 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 2, using 39 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
r2(swp2)        4      65222     599     598        0    0    0 00:29:27            3
r4(swp4)        4      65444     596     597        0    0    0 00:29:26            1
```

One will see the labels that have been shared in the "IPv4 Labeled Unicast" table.

2. Show the MPLS Forwarding Equivalency Classes:

"show mpls fec"

```
10.1.1.2/32
  Label: 102, Label Index: 2
  Client list: bgp(fd 16)
10.1.1.3/32
  Label: 103, Label Index: 3
  Client list: bgp(fd 16)
10.1.1.4/32
  Label: 104, Label Index: 4
  Client list: bgp(fd 16)
10.1.1.5/32
  Label: 105, Label Index: 5
  Client list: bgp(fd 16)
```

3. Pings between LSRs:

```
r1# ping 10.1.1.5
PING 10.1.1.5 (10.1.1.5) 56(84) bytes of data.
64 bytes from 10.1.1.5: icmp_seq=1 ttl=63 time=1.57 ms
64 bytes from 10.1.1.5: icmp_seq=2 ttl=63 time=1.82 ms
```

In this demo, you will be able to ping all of the loopback addresses from any LSR.

4. LSR routing information:

"show ip route"

```
C>* 10.1.1.1/32 is directly connected, lo, 00:32:51
B>* 10.1.1.2/32 [20/0] via fe80::4638:39ff:fe00:2, swp2, label implicit-null, 00:32:49
B>* 10.1.1.3/32 [20/0] via fe80::4638:39ff:fe00:2, swp2, label 103, 00:32:48
B>* 10.1.1.4/32 [20/0] via fe80::4638:39ff:fe00:4, swp4, label implicit-null, 00:32:48
B>* 10.1.1.5/32 [20/0] via fe80::4638:39ff:fe00:2, swp2, label 105, 00:32:48
C>* 192.168.11.0/24 is directly connected, swp10, 00:32:51
```

The "implicit-null" labels indicate that this route is one hop away from the current router. This means that the current router is the Penultimate Hop Popping (PHP) router

5. Host-based verification:

"ip r s"

```
default via 10.0.2.2 dev vagrant
10.0.2.0/24 dev vagrant proto kernel scope link src 10.0.2.15
192.168.11.0/24 dev eth1 proto kernel scope link src 192.168.11.111
192.168.22.222  encap mpls 103 via 192.168.11.1 dev eth1
192.168.200.0/24 dev eth0 proto kernel scope link src 192.168.200.11
```

This will show what is currently installed within the kernel of the host.

6. Strict LSR Forwarding

The configuration example in this demo will take a relaxed label path in order to reach the destination. There is also the option of providing a strict path of LSRs to traverse.

We will remove the relaxed configuration on host1 with the following:

```
sudo ip route del 192.168.22.222/32 encap mpls 103 via inet 192.168.11.1
```

Now, we will force traffic to go through r4, r5, and then to r3 with the following:

```
sudo ip route add 192.168.22.222/32 encap mpls 104/105/103 via inet 192.168.11.1
```

We now see the following in the kernel:

"ip r s"

```
default via 10.0.2.2 dev vagrant
10.0.2.0/24 dev vagrant proto kernel scope link src 10.0.2.15
192.168.11.0/24 dev eth1 proto kernel scope link src 192.168.11.111
192.168.22.222  encap mpls  104/105/103 via 192.168.11.1 dev eth1
192.168.200.0/24 dev eth0 proto kernel scope link src 192.168.200.11
```

### Errata

1. To shutdown the demo, run the following command from the vx-simulation directory:

    ```vagrant destroy -f```

2. This topology was configured using the Cumulus Topology Converter found at the following URL:

    https://github.com/CumulusNetworks/topology_converter

3. The following command was used to run the Topology Converter within the vx-simulation directory:

    ```./topology_converter.py cumulus-ansible-beginner-bgp-j2.dot -c```

After the above command is executed, the following configuration changes are necessary:

4. Within ```vx-simulation/helper_scripts/auto_mgmt_network/OOB_Server_Config_auto_mgmt.sh```

The following stanza:

echo " ### Creating cumulus user ###"
useradd -m cumulus

Will be replaced with the following:

echo " ### Creating cumulus user ###"
useradd -m cumulus -m -s /bin/bash

The following stanza:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.6.3

Will be replaced with the following:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.8.4

Add the following ```echo``` right before the end of the file.

    echo " ### Adding .bash_profile to auto login as cumulus user"
    echo "sudo su - cumulus" >> /home/vagrant/.bash_profile
    echo "exit" >> /home/vagrant/.bash_profile
    echo "### Adding .ssh_config to avoid HostKeyChecking"
    printf "Host * \n\t StrictHostKeyChecking no\n" >> /home/cumulus/.ssh/config

    echo "############################################"
    echo "      DONE!"
    echo "############################################"
