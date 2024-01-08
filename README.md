# WSL2Fun
KVM and Kubernetes in WSL2

## WSL configuration

Power Shell Administrator:
cd ~
notepad.exe .\.wslconfig

```text
# Settings apply across all Linux distros running on WSL 2
[wsl2]

# Limits VM memory to use no more than 4 GB, this can be set as whole numbers using GB or MB
memory=50GB

# Sets the VM to use two virtual processors
#processors=2

# Specify a custom Linux kernel to use with your installed distros. The default kernel used can be found at https://github.com/microsoft/WSL2-Linux-Kernel
kernel=C:\\Users\\XXX\\bzImage

# Sets additional kernel parameters, in this case enabling older Linux base images such as Centos 6
#kernelCommandLine = vsyscall=emulate

# Sets amount of swap storage space to 8GB, default is 25% of available RAM
swap=4GB

# Sets swapfile path location, default is %USERPROFILE%\AppData\Local\Temp\swap.vhdx
#swapfile=C:\\temp\\wsl-swap.vhdx

# Disable page reporting so WSL retains all allocated memory claimed from Windows and releases none back when free
#pageReporting=false

#
guiApplications=true

# Turn on default connection to bind WSL 2 localhost to Windows localhost
localhostforwarding=true

# Nested virtualization
nestedVirtualization=true

# Turns on output console showing contents of dmesg when opening a WSL 2 distro for debugging
debugConsole=true

# Enable experimental features
[experimental]
sparseVhd=true
```

## WSL2 basic commands

wsl -l -v
```text
  NAME                   STATE           VERSION
* docker-desktop-data    Running         2
  Ubuntu                 Running         2
  docker-desktop         Running         2
```

Docker Desktop conflicts with KVM, disable automatic start from Docker Desktop settings and reboot: 

```text
C:\Users\XXX>wsl -l -v
  NAME                   STATE           VERSION
* docker-desktop-data    Stopped         2
  Ubuntu                 Running         2
  docker-desktop         Stopped         2
```

wsl --shutdown does not work for DD 

Start Ubuntu from cmd prompt: wsl -d Ubuntu


## WSL2 default Ubuntu configuration

\# apt install cpu-checker
```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  msr-tools
The following NEW packages will be installed:
  cpu-checker msr-tools
0 upgraded, 2 newly installed, 0 to remove and 1 not upgraded.
Need to get 17.1 kB of archives.
After this operation, 67.6 kB of additional disk space will be used.
Do you want to continue? [Y/n] Y
```

\# kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used

$ sudo apt  install aria2 make build-essential libncurses-dev bison flex libssl-dev libelf-dev bc dwarves


## Customize Linux Kernel

Download latest kernel:
$ aria2c https://github.com/microsoft/WSL2-Linux-Kernel/archive/refs/tags/linux-msft-wsl-5.15.133.1.tar.gz

$ tar xvfz WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1.tar.gz

Default kernel configs:
```text
ls -la Microsoft/
total 8
drwxr-xr-x  2 foofoo foofoo 4096 Oct  7 02:23 .
drwxr-xr-x 27 foofoo foofoo 4096 Oct  7 02:23 ..
lrwxrwxrwx  1 foofoo foofoo   30 Oct  7 02:23 config-wsl -> ../arch/x86/configs/config-wsl
lrwxrwxrwx  1 foofoo foofoo   38 Oct  7 02:23 config-wsl-arm64 -> ../arch/arm64/configs/config-wsl-arm64
```

/kernel/WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1$ cp Microsoft/config-wsl ./.config

Customize kernel config:

```text
$ make menuconfig
Virtualization => KVM for Intel processors support => 'M'
Processor type and features => Linux guest support => '*' KVM Guest support
Generic IOMMU Pagetable Support
```

```text
$ grep 'KVM' .config
CONFIG_KVM_GUEST=y
CONFIG_HAVE_KVM=y
CONFIG_HAVE_KVM_IRQCHIP=y
CONFIG_HAVE_KVM_IRQFD=y
CONFIG_HAVE_KVM_IRQ_ROUTING=y
CONFIG_HAVE_KVM_EVENTFD=y
CONFIG_KVM_MMIO=y
CONFIG_KVM_ASYNC_PF=y
CONFIG_HAVE_KVM_MSI=y
CONFIG_HAVE_KVM_CPU_RELAX_INTERCEPT=y
CONFIG_KVM_VFIO=y
CONFIG_KVM_GENERIC_DIRTYLOG_READ_PROTECT=y
CONFIG_KVM_COMPAT=y
CONFIG_HAVE_KVM_IRQ_BYPASS=y
CONFIG_HAVE_KVM_NO_POLL=y
CONFIG_KVM_XFER_TO_GUEST_WORK=y
CONFIG_KVM=y
CONFIG_KVM_WERROR=y
CONFIG_KVM_INTEL=m
CONFIG_KVM_AMD=y
\# CONFIG_KVM_XEN is not set
\# CONFIG_KVM_MMU_AUDIT is not set
CONFIG_PTP_1588_CLOCK_KVM=y
```

$ make -j 8
...
Kernel: arch/x86/boot/bzImage is ready  (#2)

$ sudo make modules_install
arch/x86/Makefile:142: CONFIG_X86_X32 enabled but no binutils support
  INSTALL /lib/modules/5.15.133.1-microsoft-standard-WSL2/kernel/arch/x86/kvm/kvm-intel.ko
  DEPMOD  /lib/modules/5.15.133.1-microsoft-standard-WSL2

~/kernel/WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1$ grep 'CONFIG_X86_X32' .config
CONFIG_X86_X32=y <= TURN OFF NEXT TIME !!!

Creates /lib/modules/KERNEL_VERSION:
$ ls /lib/modules/$(uname -r)
build          modules.alias.bin          modules.builtin.bin      modules.dep.bin  modules.softdep      source
kernel         modules.builtin            modules.builtin.modinfo  modules.devname  modules.symbols
modules.alias  modules.builtin.alias.bin  modules.dep              modules.order    modules.symbols.bin

ls /lib/modules/$(uname -r)/kernel/arch/x86/kvm/
kvm-intel.ko

$ ls -la arch/x86_64/boot/bzImage
lrwxrwxrwx 1 foofoo foofoo 22 Dec  4 12:39 arch/x86_64/boot/bzImage -> ../../x86/boot/bzImage

~/kernel/WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1$ cp arch/x86/boot/bzImage /mnt/c/Users/XXX/bzImage

EDIT .wslconfig (already included in above .wslconfig example)
kernel=C:\\Users\\[WINUSER]\\bzImage

KVM module parameters:

```text
/etc/modprobe.d# cat kvm-nested.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
```

Reload wsl:
C:\Users\XXX>wsl -l -v
  NAME                   STATE           VERSION
* docker-desktop-data    Stopped         2
  Ubuntu                 Stopped         2
  docker-desktop         Stopped         2

C:\Users\rXXX>wsl --shutdown Ubuntu

C:\Users\rXXX>wsl -d Ubuntu

Module check:
\# lsmod
Module                  Size  Used by
kvm_intel             327680  0

\# kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used

\# cat /sys/module/kvm_intel/parameters/nested
Y


## Install KVM

$ sudo apt -y install  qemu-kvm virt-manager libvirt-daemon-system virtinst libvirt-clients bridge-utils

$ sudo systemctl status libvirtd
 libvirtd.service - Virtualization daemon
     Loaded: loaded (/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-12-04 13:49:03 EET; 2min 55s ago

$ sudo usermod -aG libvirt $USER
$ exit
logout

C:\Users\rXXX>wsl -d Ubuntu


Note! X for wsl2: wsl2 has now native support for GUI apps


Deploy Ubuntu nested KVM VM host:
$ aria2c https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
$ aria2c https://cloud-images.ubuntu.com/jammy/current/MD5SUMS

$ md5sum -c MD5SUMS-Ubuntu
jammy-server-cloudimg-amd64.img: OK

$ apt install cloud-init libguestfs-tools cloud-image-utils

$ cloud-init -v
/usr/bin/cloud-init 23.3.3-0ubuntu0~22.04.1

$ sudo python3 -m cloudinit.cmd.main schema -c ./user-data  ==  sudo cloud-init schema -c user-data
Valid cloud-config: ./user-data

$ cloud-localds seed1.iso ./jammy-seeds/user-data ./jammy-seeds/meta-data

$ virt-install --os-variant list

Selfmade seed:

```text
virt-install \
        --virt-type kvm \
        --connect qemu:///system \
        --name "Jammy1" \
        --description "Ubu test" \
        --ram 1024 \
        --vcpus 1 \
        --disk path="~/bakery/jammy-server-cloudimg-amd64-50G.qcow2",device=disk,bus=virtio \
        --disk path="~/bakery/Ubuntu-cloudinit/seed1.iso",device=disk,bus=virtio,readonly=on \
        --os-type linux \
        --os-variant ubuntu22.04 \
        --network bridge=virbr0 \
        --graphics none \
        --debug \
        --noreboot \
        --console pty,target_type=serial \
        --qemu-commandline='-smbios type=1' \
        --rng /dev/urandom \
        --clock offset=localtime \
        --import
```

$ virsh list --all
 Id   Name     State
-------------------------
 -    Jammy1   shut off

Change default 'root' to enable virsh start: 
/etc/libvirt/qemu.conf
user = "XXX"
group = "libvirt"

\# systemctl restart libvirtd

\# Set default DHCP range
$ virsh net-edit default
 <range start='192.168.122.100' end='192.168.122.254'/>

$ virsh net-destroy default
Network default destroyed

```text
$ virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   yes         yes

```text

$ virsh net-start default
Network default started

```text
$ virsh net-list --all
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

Virt install with Yaml seeds:

```text
virt-install \
        --virt-type kvm \
        --connect qemu:///system \
        --name "Kube1" \
        --description "Kube" \
        --ram 4096 \
        --vcpus 2 \
        --disk path="~/bakery/jammy-server-cloudimg-amd64-50G-1.qcow2",device=disk,bus=virtio \
        --cloud-init user-data="./Ubuntu-cloudinit/jammy-seeds2/user-data.yaml",meta-data="./Ubuntu-cloudinit/jammy-seeds2/meta-data.yaml",network-config="./Ubuntu-cloudinit/jammy-seeds2/network-config.yaml" \
        --os-type linux \
        --os-variant ubuntu22.04 \
        --network bridge=virbr0,model=virtio \
        --graphics none \
        --debug \
        --console pty,target_type=serial \
        --qemu-commandline='-smbios type=1' \
        --rng /dev/urandom \
        --clock offset=localtime \
        --import
```

## Deploy docker repo key

wget https://download.docker.com/linux/ubuntu/gpg -O docker.gpg.pub.key
gpg --list-packets docker.gpg.pub.key | awk '/keyid:/{ print $2 }'
8D81803C0EBFCD88
7EA0A9C3F273FCD8

Disable /etc/hosts auto generation because there was syntax error in that file:

```text
# cat /etc/wsl.conf

[boot]
systemd=true

[network]
generateHosts = false
```

## Deploy K8s with Calico CNI

kubeadm init --v=5 --ignore-preflight-errors=NumCPU,Mem --node-name=kube1 --pod-network-cidr=10.244.0.0/16 --upload-certs --control-plane-endpoint=kube1 --kubernetes-version=v1.28.2

Set K8s basics:
```text
$ mkdir -p /home/k8s-admin/.kube
$ chown  k8s-admin:k8s-admin /home/k8s-admin/.kube
$ sudo cp -i /etc/kubernetes/admin.conf /home/k8s-admin/.kube/config
$ sudo chown k8s-admin:k8s-admin /home/k8s-admin/.kube/config
```

```text
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/tigera-operator.yaml
namespace/tigera-operator created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/apiservers.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/imagesets.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
serviceaccount/tigera-operator created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
deployment.apps/tigera-operator created
```

k8s-admin@kube1:~$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml -O

```text
$ cat custom-resources.yaml => set ip range
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

k8s-admin@kube1:~$ kubectl create -f custom-resources.yaml
installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created

```text
k8s-admin@kube1:~$ kubectl get nodes -o wide
NAME    STATUS   ROLES           AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kube1   Ready    control-plane   20m   v1.28.2   192.168.122.10   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
```

```text
k8s-admin@kube1:~$ kubectl get pods --all-namespaces
NAMESPACE          NAME                                      READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-597cc87d47-dmv9p         1/1     Running   0          46s
calico-apiserver   calico-apiserver-597cc87d47-qgx6j         1/1     Running   0          46s
calico-system      calico-kube-controllers-9fd9bb574-2bwpc   1/1     Running   0          88s
calico-system      calico-node-k7dld                         1/1     Running   0          88s
calico-system      calico-typha-756766ddf6-95d78             1/1     Running   0          88s
calico-system      csi-node-driver-6kw25                     2/2     Running   0          88s
kube-system        coredns-5dd5756b68-lm96x                  1/1     Running   0          20m
kube-system        coredns-5dd5756b68-tfjp4                  1/1     Running   0          20m
kube-system        etcd-kube1                                1/1     Running   0          21m
kube-system        kube-apiserver-kube1                      1/1     Running   0          21m
kube-system        kube-controller-manager-kube1             1/1     Running   0          21m
kube-system        kube-proxy-hwcbh                          1/1     Running   0          20m
kube-system        kube-scheduler-kube1                      1/1     Running   0          21m
tigera-operator    tigera-operator-7f8cd97876-gdwwj          1/1     Running   0          3m17s
```

### Prep step for every node

```text
$ cat /etc/hosts
127.0.0.1 localhost

192.168.122.10 kube1.local kube1
192.168.122.11 kube2.local kube2
192.168.122.12 kube3.local kube3
192.168.122.13 kube4.local kube4
```

$ prepare-cloudimage-disk.sh -n jammy-server-cloudimg-amd64 -N 2

$ ./vm-script-ubuntu-cloudinit.sh -n Kube2 -d 'Worker 1' -p './jammy-server-cloudimg-amd64-50G-2.qcow2' -N '2'


### Create and deploy K8s join token

k8s-admin@kube1:~$ sudo kubeadm token create --print-join-command

k8s-admin@kube2:~$ sudo kubeadm join kube1:6443 --token ztrn06.m


### Final check

```text
$ ssh k8s-admin@192.168.122.10 kubectl get nodes -o wide
Enter passphrase for key '/home/foofoo/.ssh/id_rsa':
NAME    STATUS   ROLES           AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kube1   Ready    control-plane   72m     v1.28.2   192.168.122.10   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
kube2   Ready    <none>          15m     v1.28.2   192.168.122.11   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
kube3   Ready    <none>          6m37s   v1.28.2   192.168.122.12   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
kube4   Ready    <none>          23s     v1.28.2   192.168.122.13   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
```

