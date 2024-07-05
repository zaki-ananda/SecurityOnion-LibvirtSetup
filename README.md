# Security Onion Setup: Standalone Mode + Simulation using Two Devices (Laptops)
This is a documentation of my personal setup, where I use two laptops to do a **Security Onion Standalone Installation** in ***Laptop 1***, and **Network Traffic Simulation** in ***Laptop 2***. 

## Laptop 1 Setup
Standalone mode requires two NICs to be present, which means that you can't install it on laptop baremetal (since most laptop only has one NIC). Therefore, you have to use a VM to create an instance with two NICs. In this case, I used libvirt/QEMU hypervisor to host the installation. 

Here, two laptops is used because Security Onion has a very high system requirements for standalone installation: 
- 4 Core of CPU
- 16GB RAM
- 200GB Storage

which is quite expensive for it to be hosted on a VM insided a laptop. Therefore, I offloaded the installation to another laptop. Other than the Security Onion ISO file and the VM instance itself, it only houses the bare minimum of utilities and GUI. In this case, I'm using a Xubuntu with an additional installation of virt-manager.

I'm assuming you have Xubuntu (or any other distro) and virt-manager ready to use. Download the [ISO (12GB)](https://docs.securityonion.net/en/2.4/download.html), then create a new VM instance. Keep in mind the systems requirement mentioned above, **as the installer will actually block the installation if you have less than the requirement.**

The following steps is to create a new VM instance for Security Onion. Skip these details if you're familiar with virt-manager (or if you think the GUI is intuitive enough for you)
- Open virt-manager (or Virtual Machine Manager)
- *File* - *New Virtual Machine*
- *Connection: QEMU/KVM*; Select *"Local install ..."*; *Forward*
- *Browse* - *Browse Local* - Navigate to Security Onion ISO file; Unselect *"Automatically detect ..."* ; Type Oracle Linux on the search bar and select the top result; *Forward*
- *Memory:* 16GB or more; *CPU:* 4 or more; *Forward* 
- Change the disk image size to 200GB or more; *Forward*
- Change the VM instance name if you want; *Finish*

Next, we need to add another NIC to the VM instance. The existing NIC (created automatically) will be used for management (`vnet1`), while the new NIC will be used for sniffing (`vnet2`). To do this:
- *Edit - Preferences - Enable XML Editing*; *Close*
- Double click on newly created instance
- Click the information logo at the top
- *Add Hardware* - *Network*; *Finish*

Now, we can start the Security Onion installation. Start the VM. The first installation will set up Oracle Linux and various utility from the ISO to the VM. When finished, you will be prompted with the second, blue screen installation (which is known as *curses interface*). Click cancel first, as there will be something you need to do first. Run `nmtui` - *Edit connection*. You will see two Ethernet connection. For both of them, change the IPv4 and IPv6 configuration mode into disabled and delete any filled addresses, gateway, DNS servers, and search domains. This is because Security Onion installer expect those two interface to not have any IP address for the installation.

We can finally start the (second) installation. To go back to the menu, run `sudo SecurityOnion/setup/so-setup iso`. Continue with the installation prompt. In particular:
- Choose Standalone mode (duh!)
- Choose airgapped for faster installation (since it will rely solely on the files from the ISO)
- When prompted with IP address for the management server, select one from the `default` or `virbr0` address range. This can be accessed from:
   - Virtual Machine Manager window - Edit - Connection details - Virtual Networks - default






