# Security Onion Setup: Standalone Mode + Simulation using Two Devices (Laptops)
This is a documentation of my personal setup, where I use two laptops to do a **Security Onion Standalone Installation** in ***Laptop 1***, and **Network Traffic Simulation** in ***Laptop 2***. 

![Diagram](https://github.com/zaki-ananda/SecurityOnion-LibvirtSetup/blob/main/Security%20Onion%20-%20Two%20Laptop.drawio%20(4).png)

## 1. Laptop 1 Setup
Standalone mode requires two NICs to be present, which means that you can't install it on laptop baremetal (since most laptop only has one NIC). Therefore, you have to create a VM instance with two NICs. In this case, I used libvirt/QEMU hypervisor to host the VM. 

Here, two laptops is used because Security Onion has a very high system requirements for standalone installation: 
- 4 Core of CPU
- 16GB RAM
- 200GB Storage

which is quite expensive for it to be hosted on a VM insided a laptop. Therefore, I offloaded the installation to another laptop. Other than the Security Onion ISO file and the VM instance itself, it only houses the bare minimum of utilities and GUI. In this case, I'm using a Xubuntu with an additional installation of virt-manager.

<br>

### 1.1. VM Creation
I'm assuming you have Xubuntu (or any other distro) and virt-manager ready to use. Download the [ISO (12GB)](https://docs.securityonion.net/en/2.4/download.html), then create a new VM instance. Keep in mind the systems requirement mentioned above, **as the installer will actually block the installation if you have less than the requirement.**

The following steps is to create a new VM instance for Security Onion. Skip these details if you're familiar with virt-manager (or if you think the GUI is intuitive enough for you)
- Open virt-manager (or Virtual Machine Manager)
- *File* - *New Virtual Machine*
- *Connection: QEMU/KVM*; Select *"Local install ..."*; *Forward*
- *Browse* - *Browse Local* - Navigate to Security Onion ISO file; Unselect *"Automatically detect ..."* ; Type Oracle Linux on the search bar and select the top result; *Forward*
- *Memory:* 16GB or more; *CPU:* 4 or more; *Forward* (Be sure to leave some memory and CPU for the host machine)
- Change the disk image size to 200GB or more; *Forward*
- Change the VM instance name if you want; *Finish*

Next, we need to add another NIC to the VM instance. The existing NIC (created automatically) will be used for management (`vnet1`), while the new NIC will be used for sniffing (`vnet2`). To do this:
- *Virtual Machine Manager window - Edit - Connection details - Virtual Networks* - Click plus symbol to add new network - Name the network - *Finish*
- Double click on newly created instance
- Click the information logo at the top
- *Add Hardware* - *Network* - Choose newly added network - *Finish*

<br>

### 1.2. Security Onion Installation

Now, we can start the Security Onion installation. Start the VM. The first installation will set up Oracle Linux and various utility from the ISO to the VM. When finished, you will be prompted with the second installation with blue screen (known as *curses interface*). Select *Cancel* first, as there will be something you need to do first. 

Run `nmtui` - *Edit connection*. You will see two Ethernet connection. For both of them, change the IPv4 and IPv6 configuration mode into disabled and delete any filled addresses, gateway, DNS servers, and search domains. This is because Security Onion installer expect those two interface to not have any IP address yet for the installation. If you are confused on how to navigate `nmtui`, you can move around using arrow buttons, `Enter`, `Tab`, and `Esc`.

We can finally start the (second) installation. To go back to the menu, run `sudo SecurityOnion/setup/so-setup iso`. Continue with the installation prompt. In particular:
- Choose Standalone mode (duh!)
- Choose airgapped for faster installation (since it will rely solely on the files from the ISO)
- When prompted with IP address for the management server, select one from the `default` or `virbr0` address range. This can be viewed on:
   - Virtual Machine Manager window - Edit - Connection details - Virtual Networks - default
- The last prompt will ask you a single IP address or a subnet in CIDR form. Use the `default`/`virbr0` address range from earlier.

Then we wait. The second installation can take a long time, possibly more than an hour.

When you've finished your installation, this is what we've built so far
![Diagram](https://github.com/zaki-ananda/SecurityOnion-LibvirtSetup/blob/main/Security%20Onion%20-%20Two%20Laptop-Page-2.drawio%20(2).png)

<br>

### 1.3. Network Setup
Now we configure the network on Laptop 1 (the host itself, not the Security Onion VM). We have three things that we need to accomplish here:
#### 1.3.1. Turn off `enp1s0` IP addressing
This is likely optional, but still included to show that `enp1s0` requires no IP addressing for this setup to work.
```
   $ nmcli con mod 'Wired connection 1' ipv4.method disabled ipv6.method disabled
   $ nmcli con up 'Wired connection 1'
```

If you encounter any error, check the name for the connection with `nmcli con show` and find the one associated with your physical ethernet port.

#### 1.3.2. Create `vlan100` and connect it to `virbr0`
This VLAN will be used for management network, so that any host connected here will be able to access Security Onion Console 
```
   $ nmcli con add con-name vlan100 ifname vlan100 type vlan dev enp1s0 id 100 master virbr0 ipv4.method disabled ipv6.method disabled 
```

In particular, change `enp1s0` to the name of the physical ethernet interface on Laptop 2.

#### 1.3.3. Create `vlan300` and connect it to `vnet2` (sniffing interface)
This VLAN will be used for sniffing network. Keep in mind that this setup tries to emulate SPAN port forwarding. In particular, **any interface involved must not have IP addressing**, since the whole thing runs on OSI Layer 2 (data-link layer). 

Be sure to check the actual sniffing interface name on: VM details - second NIC - XML tab. The following assumes that the name of the sniffing interface is `vnet2`
```
   $ nmcli con add con-name vlan300 ifname vlan300 type vlan dev enp1s0 id 300 ipv4.method disabled ipv6.method disabled
   # ebtables -A FORWARD -i vlan300 -o vnet1 -j ACCEPT #WARNING: This is still non-persistent!
```

<br>

By this point, we've finished our configuration for Laptop 1
![Diagram](https://github.com/zaki-ananda/SecurityOnion-LibvirtSetup/blob/main/Security%20Onion%20-%20Two%20Laptop-Page-3.drawio.png)

## 2. Laptop 2 Setup
In Laptop 2, we'll accomplish two goals: setting up the management network in order to access the Security Onion Console, and creating a network simulation consisting of two hosts connected to a SPAN-capable switch. Luckily for us, Laptop 2 doesn't have a high-spec system requirement, as it only need to be capable of hosting two (relatively lightweight) VM instance at the same time. Do note that I'm assuming you have a baremetal installation of Linux for Laptop 2 ready to go, as using Windows is beyond the scope of this documentation.

## 2.1. Seeting Up Management Network
In Laptop 1, we've created VLAN 100 for the management network. Here, we'll connect the Laptop 2 userspace to said VLAN. This can be done with the following command:
```
   $ nmcli con add con-name vlan100 ifname vlan100 type vlan dev eno1 id 100 ipv4.method manual ipv6.method disabled ipv4.address '192.168.60.4/24'
```
In particular, change `eno1` to the interface name you have, and `192.168.60.4/24` to whatever IP you've set up in the Security Onion second installation (ie: valid address inside `virbr0` subnet).

You might also want to disable the wired interface IP addressing. 
```
   $ nmcli con mod 'Wired connection 1' ipv4.method disabled ipv6.method disabled
   $ nmcli con up 'Wired connection 1'
```

After that, try to access the management console via browser. If you've forgotten the IP address of the Security Onion installation, you can view it from the VM instance itself and running `ip addr` (there should be only one wired interface with an IP address, as mentioned before).

## 2.2. Creating Network Simulation
In Laptop 1, we've created VLAN 300 for sniffing network. In Laptop 1, we'll create a network simulation that it can sniff from. There are three things that we need to accomplish here:

### 2.2.1. Creating Network Switch
Here, I'm using OpenvSwitch in order to emulate a SPAN-capable network switch. This is done by the following command:
```
   # ovs-vsctl add-br vsbr0
   # ovs-vsctl add-port vsbr0 eno1
   # ovs-vsctl add-port vsbr0 vlan300
   # ovs-vsctl -- --id=@eno1 get port eno1 --id=@m create mirror so-monitor select_all=true output_port=@eno1 -- add bridge vsbr0 mirrors @m
```

Once again, replace `eno1` with the actual wired interface you have. For troubleshooting purpose, it might be good to familiarize yourself with the following command:
```
   # ovs-vsctl show
   # ovs-vsctl list bridge
   # ovs-vsctl list port
   # ovs-vsctl list mirror
   # ovs-vsctl add <table> <record> <column> <value>
   # ovs-vsctl remove <table> <record> <column> <value>
   # ovs-vsctl clear <table> <record> <column>
   # ovs-vsctl set <table> <record> <column>
   # ovs-vsctl get <table> Mrecord> <column>
```
More information can be obtained [here](https://backreference.org/2014/06/17/port-mirroring-with-linux-bridges/index.html) and [here](https://www.man7.org/linux/man-pages/man8/ovs-vsctl.8.html)

### 2.2.2. Creating Network Host
I recommend using a Kali Linux Live ISO as hosts, so that they won't need any persistent storage on the Laptop 2's storage drive, and so that you can simulate actual attack scenarios using pre-installed tools. I'm also assuming that you're using virt-manager (or libvirt) to set up the network host. I won't go over the installation process here (as it should be intuitive enough, if you've managed to reach here), but one thing to note is that **you need to change the XML config for each of the VM instance's NIC, so that it can connect to the OpenvSwitch network bridge**. More information can be seen [here](https://docs.openvswitch.org/en/latest/howto/libvirt/). For virt-manager GUI, you might need to enable XML editing first (Main window - Edit - Preferences - Enable XML editing)






