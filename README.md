# WAT

Pre-Build an ESXi image so it can be used for openstack as an VM with glance.

Hint: forked to remove author-related ssh keys, passwords and cleanup the image. Still all the credits to
virt-lighting for doing the hard work!

# Requirements

- A Linux system
- Libvirt, virt-install and virt-viewer (and qemu/firewalld depending on your system)
- CDRtools (genisoimage)
- ESXi ISO image

## Arch setup

```bash
sudo pacman -Syu ebtables dnsmasq libvirt qemu virt-install virt-viewer cdrtools firewalld

sudo systemctl enable --now firewalld
sudo systemctl restart libvirtd
```

# Build your image

Download the ISO's from the vmware interface and place it locally into the repo.

```shell
./build.sh isos/VMware-VMvisor-Installer-6.5.0.update01-5969303.x86_64.iso
./build.sh isos/VMware-VMvisor-Installer-6.7.0-8169922.x86_64.iso
./build.sh isos/VMware-VMvisor-Installer-6.7.0.update03-14320388.x86_64.iso
./build.sh isos/VMware-VMvisor-Installer-7.0U3-18644231.x86_64.iso
```

# Credentials

User: `root`
Password: `Verysecret123!!`

Obviously change those. You can also use `password` or `admin_pass` in the `user-data` to set the password
via `cloud-init`

# KVM configuration

You need to add the two following lines in kvm module configuration:

```shell
options kvm_intel nested=1 enable_apicv=n
options kvm ignore_msrs=1
```

Depending on the Linux distribution, the configuration file is located here:

- Fedora: /etc/modprobe.d/kvm.conf
- Ubuntu: /etc/modprobe.d/qemu-system-x86.conf

# OpenStack

You need to associate a couple of extra parameters to the image:

- `hw_disk_bus=sata`: if you use a old OpenStack version, you may encouter [LP1759420](https://bugs.launchpad.net/nova/+bug/1759420). The ESXi starts but cannot see its harddrive. In this case, use `ide` instead. It will slow the start up a bit, but everything will be functional..
- `hw_cpu_policy=dedicated`: this one is a nice to have to avoid Purple Screen of Death
- `img_config_drive=mandatory`: the [esxi-cloud-init.py](https://github.com/goneri/esxi-cloud-init) script within the image only support config drive.
- `hw_cdrom_bus=ide`
- `hw_vif_model=e1000`: there is no `virtio` driver on ESXi.
- `hw_qemu_guest_agent=no`: and the image does not include any QEMU agent.

You can use the following command to upload you image:

```shell
openstack image create --disk-format qcow2 --container-format bare --file esxi-${VERSION}.qcow2 --property hw_disk_bus=sata --property hw_cpu_policy=dedicated --property img_config_drive=mandatory --property hw_cdrom_bus=ide --property hw_vif_model=e1000 --property hw_boot_menu=true --property hw_qemu_guest_agent=no --min-disk 1 --min-ram 4096 esxi-${VERSION}
```

Be sure to use `hw_vif_model=e1000e` if it is ESXi 7.x
