
# Network Boot of RHEL Image Mode
This project shows how to setup a PXE Server that supports PXE
installations of a RHEL Image Mode bootable container image. This server
also runs a lightweight registry for RHEL Image Mode installations. This
project envisions a combined server that is running services to support
PXE booting of RHEL Image Mode including:

* dhcpd - provides IP address and "next server" to complete PXE boot process
* tftpd - supports legacy PXE boot via TFTP protocol
* httpd - supports UEFI PXE with HTTP and hosts the kickstart file to install from the registry
* container registry - serves the bootable container image for the installation

The same server is also used to build a bootable container image that
can be installed on a target edge device via PXE boot.

There's also a great [article](https://developers.redhat.com/articles/2024/08/20/bare-metal-deployments-image-mode-rhel)
that discusses this approach.

## Prepare your network for a new DHCP server

#### PJ's Comment:
---

> You can do this, but it seems redundant. The way he has setup the
> demo.conf tries to pull the IP from your first active ethernet
> connection IP. The problem is, for most of us at GTRI, you need the
> DHCP service to get internet in the first place, we have no way to
> reliably (see 'ever') to get an individual outside IP. So, we will
> skip this step.

---

Since you're creating a new DHCP server, you need to make sure that there
is no competing DHCP server on the target network. How to do this really
depends on your environment. I ran this as a guest VM using libvirt on
a RHEL laptop.

First, stop the default network.

    sudo virsh net-destroy default

Edit the default network configuration.

    sudo virsh net-edit default

Then delete the `<dhcp ... />` stanza and save the file. On my
installation, I removed the following lines.

    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>

Next, restart the virtual network for the changes to take effect.

    sudo virsh net-start default

Finally, check that the virtual network is running.

    sudo virsh net-info default

The virtual network should show as active.

## Start with minimal RHEL 9.4 installation
Start with a minimal install of RHEL 9.4 either on baremetal or on a
guest VM. From this point forward, we'll refer to this system as the
SERVER. Use UEFI firmware, if able to, when installing your system. Also
make sure there's sufficient disk space on the SERVER to support the
demo. I typically configure a 128 GiB disk on the SERVER.

Make sure to enable FIPS mode during installation, so that configuring
FIPS on any bootable container you create also uses properly defined
keys. To do that, select `Install Red Hat Enterprise Linux 9.4` on
first boot within the GRUB boot menu and then press `e` to edit the
boot commandline. Add `fips=1` to the end of the line that begins with
`linuxefi` and then press CTRL-X to continue booting.

During RHEL installation, configure a regular user with `sudo` privileges
on the host.

You'll also need to manually configure a static IP address for the SERVER
as it will be the DHCP server for it's network. The network settings
should make sense for the network being used. When manually configuring
my network connection during installation, I used the following network
settings based on my libvirt network.

| Parameter | Value |
| --------- | ----- |
| Configuration | Manual |
| IP Address | 192.168.122.2 |
| Subnet Mask | 255.255.255.0 |
| Default Router | 192.168.122.1 |
| DNS Server | 192.168.122.1 |

## Prepare the SERVER
These instructions assume that this repository is cloned or copied to
your user's home directory on the SERVER (e.g. `~/bootc-pxe-server`). The
below instructions follow that assumption.

Edit the `demo.conf` file and make sure the settings are correct. At a
minimum, you should adjust the credentials for simple content access.
The full list of options in the `demo.conf` file are shown here.

#### PJ's Comment:
---
> For these settings, we will tweak only a handful of things. 
> 
> SCA_USER and SCA_PASS should be your independent developer login
> credentials for Red Hat. Become a RHEL dev here:
> https://developers.redhat.com/?intcmp=701f20000012k6JAAQ 
> 
> For EDGE_USER and EDGE_PASS, I am not certain if these need to be
> changed but I did change them to the target's computer's root login
> credentials. 
> 
> Lastly under DHCP settings, we need to use the independent IP we setup
> on our internal network for the Server that this project will live on.
> The IP should be something that is accessible to both systems without
> outside conflicts. For example, internal IP range may be a
> 10.XXX.XXX.XXX for the net, we want a 192.168.XXX.XXX that is useable set to the interface being used between the Server and Target
> machines. 
> 
> In short, replace:
> `$(ip route get 8.8.8.8 | awk '{print $7; exit}') `
> 
> With the IP you chose for your internal IP address:
> `192.168.XXX.XXX`
> 
> The rest of the scripts use the `demo.conf` file to setup the internal
> DHCP network.
---

#### Red Hat Simple Content Access
| Parameter | Description |
| --------- | ----------- |
| SCA_USER | Your username |
| SCA_PASS | Your password |

#### Target Edge Device
| Parameter | Description |
| --------- | ----------- |
| EDGE_USER | User name |
| EDGE_PASS | Plaintext password |
| EDGE_HASH | SHA-512 hash of the EDGE_PASS parameter |

#### DHCP Settings
| Parameter | Description |
| --------- | ----------- |
| HOSTIP      | The routable IP address to the PXE server |
| SUBNET      | The first three tuples of the IPv4 address of the subnetwork for the PXE server |
| SUBNET_MASK | The subnet mask (e.g. 255.255.255.0) for the PXE server |
| SUBNET_IP   | The network address for the subnetwork |
| ROUTER_IP   | The IP address for the default router |

#### Bootable Container Image and Container Registry
| Parameter | Description |
| --------- | ----------- |
| BOOT_ISO         | Minimal boot ISO to extract kernel and initramfs to support PXE boot |
| REGISTRYPORT     | The port for the local container registry |
| CONTAINER_REPO   | The fully qualified name for your bootable container repository |
| REGISTRYINSECURE | Boolean for whether the registry requires TLS |
| BOOTC_KICKSTART  | The kickstart file to send to the PXE client |

#### PJ's Comment:
---
> The following boot.iso file requires that RHEL Developer account
> aforementioned.
---

Make sure to download the `BOOT_ISO` file, e.g.
[rhel-9.4-x86_64-boot.iso](https://access.redhat.com/downloads/content/rhel)
to the local copy of this repository on the SERVER
(e.g. ~/bootc-pxe-server). Run the following script to register and
update the system.

    sudo ./register-and-update.sh
    sudo reboot

#### PJ's Comment:
---

> Unless otherwise noted past this point, use `sudo su` before running
> the following commands.

---
### Install tooling to build a bootable container
The network installation supports RHEL Image Mode, aka bootable
containers. Install the tooling necessary to support creating a bootable
container image on the SERVER.

    sudo ./config-bootc.sh

### Configure a local image registry
During the network install, the client will need to pull the bootable
container image from a registry. This can be a secure registry, but for
this example we'll install a insecure local image registry on the SERVER.

    sudo ./config-registry.sh

### Build the bootable container image
In order to build the bootable container image, you'll need to pull the
base image from the Red Hat registry. Use your Red Hat credentials to
login to the registry so the later build can pull the base image.

    . demo.conf
    podman login --username $SCA_USER --password $SCA_PASS registry.redhat.io

We're now ready to build a bootable container image and push it to
the local image registry. This bootable container image will be later
installed on the client edge device. The container image is tagged as
"v1" so that you can later build different versions with alternative
tags. The client edge device will look for container images tagged as
"prod" for updates.

    . demo.conf
    podman build -f Containerfile -t $CONTAINER_REPO:v1 .
    podman push $CONTAINER_REPO:v1

Tag the container image as "prod" to indicate its the production
container image.

    podman tag $CONTAINER_REPO:v1 $CONTAINER_REPO:prod
    podman push $CONTAINER_REPO:prod

#### PJ's Comment:
---
> ./gen-ks.sh is the one script you will use `exit` and run with
> normally without root, i.e. `sudo su`.
---
### Create a kickstart for automated network installations
Now, create the kickstart file that will be used to automate the
installation of the client edge device via the network installation. Of
note is the `ostreecontainer` directive in the generated kickstart
file, which will pull content for the client edge device from the local
container registry.

    ./gen-ks.sh

Review the generated file using the following command:

    . demo.conf
    less $BOOTC_KICKSTART

#### PJ's Comment:
---
> Continue using root past this point, `sudo su`.
---
### Create web server for kickstart and UEFI HTTP installs
Set up a simple web server to host the kickstart file you previously
generated as well as the kernel and initial ram disk for network
installations. This server also supports UEFI HTTP client installations.

    sudo ./config-httpd.sh

### Create the DHCP server
The SERVER acts as the DHCP server for its local area network, providing
IP addresses, routers, DNS servers, and other information including
network boot parameters to its clients.

    sudo ./config-dhcpd.sh

### Create the TFTP server for PXE boot clients
The last thing we configure is the TFTP server for legacy PXE boot
clients. This takes content from the RHEL 9.4 boot ISO to provide the
kernel and initial ram disk to the PXE clients.

    sudo ./config-tftp.sh

## Review the setup
Now that we've installed the services necessary for PXE and UEFI HTTP
client network boots, we can review all the files that were created and
make sure that the services are running.

First, let's make sure the image registry is running and that the bootable
container image is available for network installations.

    systemctl status local-registry.service

You should see that the service is `active`. Check that the bootable
container image repository is available for installations.

    . demo.conf
    curl -s http://$HOSTIP:$REGISTRYPORT/v2/_catalog

You should see the bootable container image repository listed.

Next, check the DHCP server configuration.

    sudo cat /etc/dhcp/dhcpd.conf

You should see configuration set for both PXE and UEFI HTTP clients.

Confirm the web server is running and has the RHEL boot content to
support an network installation.

    systemctl status httpd

The service should be `active`. You can also review that the RHEL boot
content is available to clients as well.

    find /var/www/html

You should see the kickstart file under `/var/www/html/` and the RHEL
network boot content under `/var/www/html/redhat/`.

Finally, you can review the RHEL network boot content for legacy PXE
clients using the following command:

    find /var/lib/tftpboot

You can also review the GRUB boot menu entries here that will install
the guest operating system.

    cat /var/lib/tftpboot/redhat/EFI/BOOT/grub.cfg
