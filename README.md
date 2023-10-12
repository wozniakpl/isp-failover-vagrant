# ISP Failover with Vagrant

This repository provides a Vagrant setup to simulate ISP failover. It consists of three virtual machines: two ISPs (`isp1` and `isp2`) and a client. The client is configured to switch between the two ISPs based on their availability.

## Prerequisites

- [Vagrant](https://www.vagrantup.com/)
- [VirtualBox](https://www.virtualbox.org/)

## Setup

1. Clone this repository:
   ```bash
   git clone https://github.com/wozniakpl/isp-failover-vagrant
   cd isp-failover-vagrant
   ```

2. Start the Vagrant environment:
   ```bash
   vagrant up
   ```

3. Once the VMs are up, you can SSH into any of them using:
   ```bash
   vagrant ssh <vm-name>
   ```
   Replace `<vm-name>` with `isp1`, `isp2`, or `client` based on which VM you want to access.

## How It Works

- Both `isp1` and `isp2` VMs run a probe script that checks their connectivity to the internet (by pinging `8.8.8.8`). They then broadcast their status on port `1500` using `netcat`.

- The `client` VM runs a failover script that listens to the status of both ISPs. Based on the status received, it adjusts its default route to use the ISP that's currently up.

- The failover script on the client checks the status of the ISPs every second and adjusts the routes accordingly.

- The `traceroute` tool is installed on the client VM for debugging purposes.

### Network Interfaces

#### enp0s8 (Private Network Interface)

In the Vagrant setup, `enp0s8` is a private network interface. This interface allows VMs within the same Vagrant environment to communicate with each other on a private subnet. It's isolated from the host machine and the external world, making it ideal for inter-VM communications without any external interference.

In our setup:
- The `client` VM uses this interface to communicate with the `isp1` and `isp2` VMs.
- The `isp1` and `isp2` VMs have their private IP addresses (`192.168.56.101` and `192.168.56.102` respectively) on this interface.

#### enp0s9 (Bridged Interface)

The `enp0s9` interface is a bridged network interface. A bridged interface connects a virtual machine to a network by using the network adapter on the host system. This means that the VM appears as a separate device on the same physical network as the host, allowing it to have direct access to the external network and, in this case, the Internet.

In our setup:
- Both `isp1` and `isp2` VMs use this interface to access the Internet.
- The NAT (Network Address Translation) rule set up with `iptables` on the ISPs uses this interface (`enp0s9`) to masquerade the traffic from the `client` VM, allowing the client to access the Internet via the ISPs.

### Deleting the Default Route via 10.0.2.2

In a typical Vagrant setup with VirtualBox, the default NAT interface (`enp0s3` in many cases) is used to provide the VM with access to the external world. The default gateway for this interface is usually `10.0.2.2`.

However, in our setup, we want to control the Internet access of our VMs through our custom-defined ISPs (`isp1` and `isp2`). To ensure that the traffic from the `client` VM goes through our ISPs and not directly through the default VirtualBox NAT, we delete the default route via `10.0.2.2`.

By doing this:
- We ensure that the `client` VM doesn't have direct Internet access via the default VirtualBox NAT.
- All traffic from the `client` VM is routed through our custom ISPs, allowing us to simulate the ISP failover scenario effectively.


## Testing Failover

1. SSH into the client VM:
   ```bash
   vagrant ssh client
   ```

2. Start a continuous ping to an external IP, e.g., `8.8.8.8`:
   ```bash
   ping 8.8.8.8
   ```

3. In a separate terminal, SSH into one of the ISP VMs and disable its public interface:
   ```bash
   vagrant ssh isp1
   sudo ip link set enp0s9 down
   ```

4. Observe the client's ping. After a brief moment, it should switch to using the other ISP.

5. To bring the ISP back up:
   ```bash
   sudo ip link set enp0s9 up
   ```

## Cleanup

To stop the VMs and clean up the environment:
```bash
vagrant halt
vagrant destroy
```
