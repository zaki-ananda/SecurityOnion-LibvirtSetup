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
```
   $ nmcli con mod 'Wired connection 1' ipv4.method disabled ipv6.method disabled
   $ nmcli con up 'Wired connection 1'
```

If you encounter any error, check the name for the connection with `nmcli con show` and find the one associated with your physical ethernet port.

#### 1.3.2. Create `vlan100` and connect it to `virbr0`
```
   $ nmcli con add con-name vlan100 ifname vlan100 type vlan dev enp1s0 id 100 master virbr0 ipv4.method disabled ipv6.method disabled 
```
#### 1.3.3. Create `vlan300` and connect it to `vnet2` (sniffing interface)
Be sure to check the actual sniffing interface name on: VM details - second NIC - XML tab. The following assumes that the name of the sniffing interface is `vnet2`
```
   $ nmcli con add con-name vlan300 ifname vlan300 type vlan dev enp1s0 id 300 ipv4.method disabled ipv6.method disabled
   $ sudo ebtables -A FORWARD -i vlan300 -o vnet1 -j ACCEPT #WARNING: This is still non-persistent!
```

<br>

By this point, we've finished our configuration for Laptop 1
![Diagram](https://github.com/zaki-ananda/SecurityOnion-LibvirtSetup/blob/main/Security%20Onion%20-%20Two%20Laptop-Page-3.drawio.png)






