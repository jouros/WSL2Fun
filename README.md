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

wsl -v
```text
WSL version: 2.1.0.0
Kernel version: 5.15.137.3-1
WSLg version: 1.0.59
MSRDC version: 1.2.4677
Direct3D version: 1.611.1-81528511
DXCore version: 10.0.25131.1002-220531-1700.rs-onecore-base2-hyp
Windows version: 10.0.22635.3061
```

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

```text
# kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

$ sudo apt  install aria2 make build-essential libncurses-dev bison flex libssl-dev libelf-dev bc dwarves


## Customize Linux Kernel

Download latest kernel:

```text
$ aria2c https://github.com/microsoft/WSL2-Linux-Kernel/archive/refs/tags/linux-msft-wsl-5.15.133.1.tar.gz
```

```text
$ tar xvfz WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1.tar.gz
```

Default kernel configs:
```text
ls -la Microsoft/
total 8
drwxr-xr-x  2 foofoo foofoo 4096 Oct  7 02:23 .
drwxr-xr-x 27 foofoo foofoo 4096 Oct  7 02:23 ..
lrwxrwxrwx  1 foofoo foofoo   30 Oct  7 02:23 config-wsl -> ../arch/x86/configs/config-wsl
lrwxrwxrwx  1 foofoo foofoo   38 Oct  7 02:23 config-wsl-arm64 -> ../arch/arm64/configs/config-wsl-arm64
```

```text
/kernel/WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1$ cp Microsoft/config-wsl ./.config
```

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

```text
$ make -j 8
... a lot of output here
Kernel: arch/x86/boot/bzImage is ready  (#2)
```

```text
$ sudo make modules_install
arch/x86/Makefile:142: CONFIG_X86_X32 enabled but no binutils support
  INSTALL /lib/modules/5.15.133.1-microsoft-standard-WSL2/kernel/arch/x86/kvm/kvm-intel.ko
  DEPMOD  /lib/modules/5.15.133.1-microsoft-standard-WSL2
```

~/kernel/WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1$ grep 'CONFIG_X86_X32' .config
CONFIG_X86_X32=y <= TURN OFF NEXT TIME !!!

Creates /lib/modules/KERNEL_VERSION:

```text
$ ls /lib/modules/$(uname -r)
build          modules.alias.bin          modules.builtin.bin      modules.dep.bin  modules.softdep      source
kernel         modules.builtin            modules.builtin.modinfo  modules.devname  modules.symbols
modules.alias  modules.builtin.alias.bin  modules.dep              modules.order    modules.symbols.bin
```

```text
ls /lib/modules/$(uname -r)/kernel/arch/x86/kvm/
kvm-intel.ko
```

```text
$ ls -la arch/x86_64/boot/bzImage
lrwxrwxrwx 1 foofoo foofoo 22 Dec  4 12:39 arch/x86_64/boot/bzImage -> ../../x86/boot/bzImage
```

```text
~/kernel/WSL2-Linux-Kernel-linux-msft-wsl-5.15.133.1$ cp arch/x86/boot/bzImage /mnt/c/Users/XXX/bzImage
```

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

```text
C:\Users\XXX>wsl -l -v
  NAME                   STATE           VERSION
* docker-desktop-data    Stopped         2
  Ubuntu                 Stopped         2
  docker-desktop         Stopped         2
```

C:\Users\rXXX>wsl --shutdown Ubuntu

C:\Users\rXXX>wsl -d Ubuntu

Module check:

```text
# lsmod
Module                  Size  Used by
kvm_intel             327680  0
```

```text
# kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

```text
# cat /sys/module/kvm_intel/parameters/nested
Y
```

## Install KVM

```text
$ sudo apt -y install  qemu-kvm virt-manager libvirt-daemon-system virtinst libvirt-clients bridge-utils
```

```text
$ sudo systemctl status libvirtd
 libvirtd.service - Virtualization daemon
     Loaded: loaded (/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-12-04 13:49:03 EET; 2min 55s ago
```

```text
$ sudo usermod -aG libvirt $USER
$ exit
logout
```

C:\Users\rXXX>wsl -d Ubuntu

Note! X for wsl2: wsl2 has now native support for GUI apps


Deploy Ubuntu nested KVM VM host:

```text
$ aria2c https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
$ aria2c https://cloud-images.ubuntu.com/jammy/current/MD5SUMS
$ md5sum -c MD5SUMS-Ubuntu
jammy-server-cloudimg-amd64.img: OK
$ apt install cloud-init libguestfs-tools cloud-image-utils
$ cloud-init -v
/usr/bin/cloud-init 23.3.3-0ubuntu0~22.04.1
```

Check schema:

```text
$ sudo python3 -m cloudinit.cmd.main schema -c ./user-data  ==  sudo cloud-init schema -c user-data
Valid cloud-config: ./user-data
```

Create seed ISO:

```text
$ cloud-localds seed1.iso ./jammy-seeds/user-data ./jammy-seeds/meta-data
```

Check os altgernatives:

```text
$ virt-install --os-variant list
```

Virt install with Selfmade seed ISO:

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

```text
$ virsh list --all
 Id   Name     State
-------------------------
 -    Jammy1   shut off
```


Change default 'root' to enable virsh start in /etc/libvirt/qemu.conf

```text
user = "XXX"
group = "libvirt"
```

Restart Libvirt:

```text
# systemctl restart libvirtd
```

Set default DHCP range for KVM:

```text
$ virsh net-edit default
 <range start='192.168.122.100' end='192.168.122.254'/>

$ virsh net-destroy default
Network default destroyed
```

```text
$ virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   yes         yes
````

```text
$ virsh net-start default
Network default started
```

```text
$ virsh net-list --all
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

Virt install with Yaml seeds, every node has its own cloudinit config folder, below is first worker node 'kube2' deployment:

```text
virt-install \
        --virt-type kvm \
        --connect qemu:///system \
        --name "Kube2" \
        --description "Kube" \
        --ram 4096 \
        --vcpus 2 \
        --disk path="~/bakery/jammy-server-cloudimg-amd64-50G-2.qcow2",device=disk,bus=virtio \
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

## Deploy Docker repo key

```text
$ wget https://download.docker.com/linux/ubuntu/gpg -O docker.gpg.pub.key
gpg --list-packets docker.gpg.pub.key | awk '/keyid:/{ print $2 }'
8D81803C0EBFCD88
7EA0A9C3F273FCD8
```

Disable /etc/hosts auto generation because there was syntax error in that file:

```text
# cat /etc/wsl.conf

[boot]
systemd=true

[network]
generateHosts = false
```

## Deploy K8s with Calico CNI

```text
kubeadm init --v=5 --ignore-preflight-errors=NumCPU,Mem --node-name=kube1 --pod-network-cidr=10.244.0.0/16 --upload-certs --control-plane-endpoint=kube1 --kubernetes-version=v1.28.2
```

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

```text
k8s-admin@kube1:~$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml -O
```

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

```text
k8s-admin@kube1:~$ kubectl create -f custom-resources.yaml
installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created
```

Control-plane is now ready:

```text
k8s-admin@kube1:~$ kubectl get nodes -o wide
NAME    STATUS   ROLES           AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kube1   Ready    control-plane   20m   v1.28.2   192.168.122.10   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
```

Calico CNI is ready:

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

### Prep steps for every node

Set /etc/hosts for every node:

```text
$ cat /etc/hosts
127.0.0.1 localhost

192.168.122.10 kube1.local kube1
192.168.122.11 kube2.local kube2
192.168.122.12 kube3.local kube3
192.168.122.13 kube4.local kube4
```

Prepare VM images for deployments one by one:

```text
$ prepare-cloudimage-disk.sh -n jammy-server-cloudimg-amd64 -N 2
```

Create worker nodes one by one:

```text
$ ./vm-script-ubuntu-cloudinit.sh -n Kube2 -d 'Worker 1' -p './jammy-server-cloudimg-amd64-50G-2.qcow2' -N '2'
```

Every VM has its own qcow2 disk image:

```text
$ virsh dumpxml Kube1 | grep 'source file'
      <source file='/home/XXX/bakery/jammy-server-cloudimg-amd64-50G-1.qcow2' index='2'/>
$ virsh dumpxml Kube2 | grep 'source file'
      <source file='/home/XXX/bakery/jammy-server-cloudimg-amd64-50G-2.qcow2' index='2'/>
$ virsh dumpxml Kube3 | grep 'source file'
      <source file='/home/XXX/bakery/jammy-server-cloudimg-amd64-50G-3.qcow2' index='2'/>
$ virsh dumpxml Kube4 | grep 'source file'
      <source file='/home/XXX/bakery/jammy-server-cloudimg-amd64-50G-4.qcow2' index='2'/>
```

### Create and deploy K8s join token

Create token:

```text
k8s-admin@kube1:~$ sudo kubeadm token create --print-join-command
```

Join worker node:

```text
k8s-admin@kube2:~$ sudo kubeadm join kube1:6443 --token ztrn06.m
```

### Final check

K8s is now up'n'running :)

```text
$ ssh k8s-admin@192.168.122.10 kubectl get nodes -o wide
Enter passphrase for key '/home/foofoo/.ssh/id_rsa':
NAME    STATUS   ROLES           AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kube1   Ready    control-plane   72m     v1.28.2   192.168.122.10   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
kube2   Ready    <none>          15m     v1.28.2   192.168.122.11   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
kube3   Ready    <none>          6m37s   v1.28.2   192.168.122.12   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
kube4   Ready    <none>          23s     v1.28.2   192.168.122.13   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.6.26
```

## Install Ansible

I just like to use Ansible for all kind of management tasks. 

```text
# apt install ansible
# exit
$ ansible --version
ansible 2.10.8
  config file = /home/XXX/WSL2Fun/ansible.cfg
  configured module search path = ['/home/XXX/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.10.12 (main, Nov 20 2023, 15:14:05) [GCC 11.4.0]
```

Test connection to K8s:

```text
$ ansible-playbook ansible-test.yaml
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Connection test] ****************************************************************************************************************************************************************************************

TASK [Check host] *********************************************************************************************************************************************************************************************
Tuesday 09 January 2024  16:00:28 +0200 (0:00:00.007)       0:00:00.007 *******
changed: [kube1]
changed: [kube4]
changed: [kube3]
changed: [kube2]

TASK [Show response] ******************************************************************************************************************************************************************************************
Tuesday 09 January 2024  16:00:29 +0200 (0:00:01.003)       0:00:01.011 *******
ok: [kube2] =>
  msg: |2-
     16:00:30 up  1:31,  0 users,  load average: 0.03, 0.03, 0.00
    Linux kube2 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
    kube2
    uid=0(root) gid=0(root) groups=0(root)
    VERSION="22.04.3 LTS (Jammy Jellyfish)"
ok: [kube1] =>
  msg: |2-
     16:00:30 up  1:31,  0 users,  load average: 0.20, 0.52, 0.45
    Linux kube1 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
    kube1
    uid=0(root) gid=0(root) groups=0(root)
    VERSION="22.04.3 LTS (Jammy Jellyfish)"
ok: [kube3] =>
  msg: |2-
     16:00:30 up  1:31,  0 users,  load average: 0.16, 0.07, 0.02
    Linux kube3 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
    kube3
    uid=0(root) gid=0(root) groups=0(root)
    VERSION="22.04.3 LTS (Jammy Jellyfish)"
ok: [kube4] =>
  msg: |2-
     16:00:30 up  1:31,  0 users,  load average: 0.00, 0.02, 0.00
    Linux kube4 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
    kube4
    uid=0(root) gid=0(root) groups=0(root)
    VERSION="22.04.3 LTS (Jammy Jellyfish)"

PLAY RECAP ****************************************************************************************************************************************************************************************************
kube1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kube2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kube3                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kube4                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Tuesday 09 January 2024  16:00:29 +0200 (0:00:00.035)       0:00:01.046 *******
===============================================================================
Check host --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 1.00s
Show response ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 0.04s
```

## Kube systems management with Ansible

Kube packages are marked as hold in cloud config user-data.yaml

```text
root@kube1:/home/k8s-admin# dpkg --get-selections | grep kube
kubeadm                                         hold
kubectl                                         hold
kubelet                                         hold
kubernetes-cni                                  install
```

Upgrade:

```text
$ ansible-playbook main.yml --tags "setup"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Linux host setup] ************************************************************************************************************************************************************

TASK [setup : Allow release-info to change for APT repositories] *******************************************************************************************************************
Tuesday 09 January 2024  16:25:52 +0200 (0:00:00.007)       0:00:00.007 *******
[WARNING]: Consider using the apt module rather than running 'apt-get'.  If you need to use command because apt is insufficient you can add 'warn: false' to this command task or
set 'command_warnings=False' in ansible.cfg to get rid of this message.
changed: [kube1]
changed: [kube2]
changed: [kube3]
changed: [kube4]

TASK [setup : Upgrade all apt packages] ********************************************************************************************************************************************
Tuesday 09 January 2024  16:25:57 +0200 (0:00:04.973)       0:00:04.981 *******
changed: [kube1]
changed: [kube2]
changed: [kube3]
changed: [kube4]

TASK [setup : Show current kernel] *************************************************************************************************************************************************
Tuesday 09 January 2024  16:26:04 +0200 (0:00:07.173)       0:00:12.154 *******
changed: [kube4]
changed: [kube3]
changed: [kube2]
changed: [kube1]

TASK [setup : debug] ***************************************************************************************************************************************************************
Tuesday 09 January 2024  16:26:05 +0200 (0:00:00.160)       0:00:12.315 *******
ok: [kube1] =>
  msg: 5.15.0-91-generic
ok: [kube2] =>
  msg: 5.15.0-91-generic
ok: [kube3] =>
  msg: 5.15.0-91-generic
ok: [kube4] =>
  msg: 5.15.0-91-generic

TASK [setup : Check if a reboot is needed for Debian and Ubuntu boxes] *************************************************************************************************************
Tuesday 09 January 2024  16:26:05 +0200 (0:00:00.031)       0:00:12.346 *******
ok: [kube4]
ok: [kube2]
ok: [kube3]
ok: [kube1]

TASK [setup : Reboot the Debian or Ubuntu server] **********************************************************************************************************************************
Tuesday 09 January 2024  16:26:05 +0200 (0:00:00.193)       0:00:12.539 *******
skipping: [kube1]
skipping: [kube2]
skipping: [kube3]
skipping: [kube4]

TASK [setup : Install additional packages] *****************************************************************************************************************************************
Tuesday 09 January 2024  16:26:05 +0200 (0:00:00.031)       0:00:12.570 *******
changed: [kube2]
changed: [kube3]
changed: [kube4]
changed: [kube1]

TASK [setup : Show current kernel] *************************************************************************************************************************************************
Tuesday 09 January 2024  16:26:12 +0200 (0:00:06.887)       0:00:19.458 *******
changed: [kube2]
changed: [kube3]
changed: [kube4]
changed: [kube1]

TASK [setup : debug] ***************************************************************************************************************************************************************
Tuesday 09 January 2024  16:26:12 +0200 (0:00:00.136)       0:00:19.594 *******
ok: [kube1] =>
  msg: 5.15.0-91-generic
ok: [kube2] =>
  msg: 5.15.0-91-generic
ok: [kube3] =>
  msg: 5.15.0-91-generic
ok: [kube4] =>
  msg: 5.15.0-91-generic

PLAY RECAP *************************************************************************************************************************************************************************
kube1                      : ok=8    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
kube2                      : ok=8    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
kube3                      : ok=8    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
kube4                      : ok=8    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

Tuesday 09 January 2024  16:26:12 +0200 (0:00:00.034)       0:00:19.629 *******
===============================================================================
setup : Upgrade all apt packages -------------------------------------------------------------------------------------------------------------------------------------------- 7.17s
setup : Install additional packages ----------------------------------------------------------------------------------------------------------------------------------------- 6.89s
setup : Allow release-info to change for APT repositories ------------------------------------------------------------------------------------------------------------------- 4.97s
setup : Check if a reboot is needed for Debian and Ubuntu boxes ------------------------------------------------------------------------------------------------------------- 0.19s
setup : Show current kernel ------------------------------------------------------------------------------------------------------------------------------------------------- 0.16s
setup : Show current kernel ------------------------------------------------------------------------------------------------------------------------------------------------- 0.14s
setup : debug --------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.03s
setup : debug --------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.03s
setup : Reboot the Debian or Ubuntu server ---------------------------------------------------------------------------------------------------------------------------------- 0.03s
```

### Deploy Helm to Kube with Ansible

Helm is defacto package manager for K8s and I prefer to use it everywhere. Helm can be deployed to control-plane with:

```text
$ ansible-playbook main.yml --tags "helm-deployment"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Linux host setup] *********************************************************************************************************************************************************************************

TASK [helm-deployment : Get repo key] *******************************************************************************************************************************************************************
Wednesday 10 January 2024  11:15:12 +0200 (0:00:00.011)       0:00:00.011 *****
[WARNING]: Module remote_tmp /root/.ansible/tmp did not exist and was created with a mode of 0700, this may cause issues when running as another user. To avoid this, create the remote_tmp dir with the
correct permissions manually
changed: [kube1]

TASK [helm-deployment : Add repository to sources.list] *************************************************************************************************************************************************
Wednesday 10 January 2024  11:15:13 +0200 (0:00:00.899)       0:00:00.910 *****
changed: [kube1]

TASK [helm-deployment : Install Helm] *******************************************************************************************************************************************************************
Wednesday 10 January 2024  11:15:16 +0200 (0:00:03.382)       0:00:04.292 *****
changed: [kube1]

TASK [helm-deployment : Hold Helm] **********************************************************************************************************************************************************************
Wednesday 10 January 2024  11:15:20 +0200 (0:00:04.138)       0:00:08.430 *****
changed: [kube1]

PLAY RECAP **********************************************************************************************************************************************************************************************
kube1                      : ok=4    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Wednesday 10 January 2024  11:15:20 +0200 (0:00:00.188)       0:00:08.619 *****
===============================================================================
helm-deployment : Install Helm ------------------------------------------------------------------------------------------------------------------------------------------------------------------- 4.14s
helm-deployment : Add repository to sources.list ------------------------------------------------------------------------------------------------------------------------------------------------- 3.38s
helm-deployment : Get repo key ------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.90s
helm-deployment : Hold Helm ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.19s
```

Helm v3.13.3 was latest at a time. Package is also marked as 'hold' so this deployment will stay in that version

```text
k8s-admin@kube1:~$ helm version
version.BuildInfo{Version:"v3.13.3", GitCommit:"c8b948945e52abba22ff885446a1486cb5fd3474", GitTreeState:"clean", GoVersion:"go1.20.11"}
```


### Deploy Pods to Kube with Ansible

In this part I will deploy nginx and busybox Pods to Kube. Here we need to install additional modules for Ansible:

```text
$ ansible-galaxy collection install kubernetes.core
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Installing 'kubernetes.core:3.0.0' to '/home/joro/WSL2Fun/collections/ansible_collections/kubernetes/core'
Downloading https://galaxy.ansible.com/api/v3/plugin/ansible/content/published/collections/artifacts/kubernetes-core-3.0.0.tar.gz to /home/joro/.ansible/tmp/ansible-local-2156mb_b7jja/tmpi35l90rv
kubernetes.core (3.0.0) was installed successfully
```

Modules will be installed into ./collections path as is configured in ansible.cfg

Next we have to add repositorires for Helm, very good starting point is to add Bitnami:

```text
$ ansible-playbook main.yml --tags "helm-addrepo"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
[WARNING]: Collection kubernetes.core does not support Ansible version 2.10.8

PLAY [Linux host setup] *********************************************************************************************************************************************************************************

TASK [helm-addrepo : Add bitnami helm repo] *************************************************************************************************************************************************************
Wednesday 10 January 2024  13:33:04 +0200 (0:00:00.009)       0:00:00.009 *****
[WARNING]: Module remote_tmp /home/k8s-admin/.ansible/tmp did not exist and was created with a mode of 0700, this may cause issues when running as another user. To avoid this, create the remote_tmp
dir with the correct permissions manually
changed: [kube1]

TASK [helm-addrepo : Update repo cache to show additions] ***********************************************************************************************************************************************
Wednesday 10 January 2024  13:33:06 +0200 (0:00:01.640)       0:00:01.649 *****
ok: [kube1]

TASK [helm-addrepo : Repo list] *************************************************************************************************************************************************************************
Wednesday 10 January 2024  13:33:07 +0200 (0:00:01.422)       0:00:03.072 *****
changed: [kube1]

TASK [helm-addrepo : debug] *****************************************************************************************************************************************************************************
Wednesday 10 January 2024  13:33:08 +0200 (0:00:00.200)       0:00:03.273 *****
ok: [kube1] =>
  msg: |-
    NAME    URL
    bitnami https://charts.bitnami.com/bitnami

PLAY RECAP **********************************************************************************************************************************************************************************************
kube1                      : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Wednesday 10 January 2024  13:33:08 +0200 (0:00:00.019)       0:00:03.293 *****
===============================================================================
helm-addrepo : Add bitnami helm repo ------------------------------------------------------------------------------------------------------------------------------------------------------------- 1.64s
helm-addrepo : Update repo cache to show additions ----------------------------------------------------------------------------------------------------------------------------------------------- 1.42s
helm-addrepo : Repo list ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.20s
helm-addrepo : debug ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.02s
```

In above I had set 'REPONAME: bitnami' and 'REPOURL: "https://charts.bitnami.com/bitnami" in main.yml variables.

Now we can use Bitnami packaged nginx:

```text
$ helm search repo nginx
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/nginx                           15.6.0          1.25.3          NGINX Open Source is a web server that can be a...
```

Bitnami nginx deployment with Ansible:

```text
$ ansible-playbook main.yml --tags "helm-nginx"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
[WARNING]: Collection kubernetes.core does not support Ansible version 2.10.8

PLAY [Linux host setup] *********************************************************************************************************************************************************************************

TASK [helm-nginx : Update repo cache to show additions] *************************************************************************************************************************************************
Wednesday 10 January 2024  13:56:01 +0200 (0:00:00.009)       0:00:00.009 *****
ok: [kube1]

TASK [helm-nginx : Deploy nginx] ************************************************************************************************************************************************************************
Wednesday 10 January 2024  13:56:02 +0200 (0:00:01.394)       0:00:01.403 *****
changed: [kube1]

TASK [helm-nginx : debug] *******************************************************************************************************************************************************************************
Wednesday 10 January 2024  13:56:05 +0200 (0:00:02.636)       0:00:04.040 *****
ok: [kube1] =>
  msg: |-
    Release "nginx" does not exist. Installing it now.
    NAME: nginx
    LAST DEPLOYED: Wed Jan 10 13:56:04 2024
    NAMESPACE: test1
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    CHART NAME: nginx
    CHART VERSION: 15.6.0
    APP VERSION: 1.25.3

    ** Please be patient while the chart is being deployed **
    NGINX can be accessed through the following DNS name from within your cluster:

        nginx.test1.svc.cluster.local (port 80)

    To access NGINX from outside the cluster, follow the steps below:

    1. Get the NGINX URL by running these commands:

        export SERVICE_PORT=$(kubectl get --namespace test1 -o jsonpath="{.spec.ports[0].port}" services nginx)
        kubectl port-forward --namespace test1 svc/nginx ${SERVICE_PORT}:${SERVICE_PORT} &
        echo "http://127.0.0.1:${SERVICE_PORT}"

PLAY RECAP **********************************************************************************************************************************************************************************************
kube1                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Wednesday 10 January 2024  13:56:05 +0200 (0:00:00.020)       0:00:04.060 *****
===============================================================================
helm-nginx : Deploy nginx ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 2.64s
helm-nginx : Update repo cache to show additions ------------------------------------------------------------------------------------------------------------------------------------------------- 1.39s
helm-nginx : debug ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.02s
```

Check nginx from control-plane:

```text
$ kubectl get pods -n test1
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7975748858-w4zwj   1/1     Running   0          32s
$ kubectl get svc -n test1
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.106.91.101   <none>        80/TCP    2m40s
$ curl http://10.106.91.101
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Next I'll add Swiss army knife Busybox, this time I'll use my selfmade helm chart for which I have created chart repo in my github:

```text
$ helm repo add custom-repo https://jouros.github.io/helm-repo
"custom-repo" has been added to your repositories
$ helm repo list
NAME            URL
custom-repo     https://jouros.github.io/helm-repo
$ helm search repo busybox
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
custom-repo/busybox     0.0.1           latest          A Helm chart for Kubernetes
```

Let's add custom-repo with Ansible role 'helm-addrepo', this time I used variables 'REPONAME: custom-repo' and 'REPOURL: "https://jouros.github.io/helm-repo"'

```text
$ ansible-playbook main.yml --tags "helm-addrepo"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
[WARNING]: Collection kubernetes.core does not support Ansible version 2.10.8

PLAY [Linux host setup] *********************************************************************************************************************************************************************************

TASK [helm-addrepo : Add bitnami helm repo] *************************************************************************************************************************************************************
Wednesday 10 January 2024  16:55:14 +0200 (0:00:00.008)       0:00:00.008 *****
changed: [kube1]

TASK [helm-addrepo : Update repo cache to show additions] ***********************************************************************************************************************************************
Wednesday 10 January 2024  16:55:14 +0200 (0:00:00.712)       0:00:00.721 *****
ok: [kube1]

TASK [helm-addrepo : Repo list] *************************************************************************************************************************************************************************
Wednesday 10 January 2024  16:55:16 +0200 (0:00:01.363)       0:00:02.084 *****
changed: [kube1]

TASK [helm-addrepo : debug] *****************************************************************************************************************************************************************************
Wednesday 10 January 2024  16:55:16 +0200 (0:00:00.168)       0:00:02.253 *****
ok: [kube1] =>
  msg: |-
    NAME            URL
    bitnami         https://charts.bitnami.com/bitnami
    custom-repo     https://jouros.github.io/helm-repo

PLAY RECAP **********************************************************************************************************************************************************************************************
kube1                      : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Wednesday 10 January 2024  16:55:16 +0200 (0:00:00.020)       0:00:02.274 *****
===============================================================================
helm-addrepo : Update repo cache to show additions ----------------------------------------------------------------------------------------------------------------------------------------------- 1.36s
helm-addrepo : Add bitnami helm repo ------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.71s
helm-addrepo : Repo list ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.17s
helm-addrepo : debug ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.02s
```

Now everyhting is ready for Busybox deployment:

```text
$ ansible-playbook main.yml --tags "helm-busybox"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
[WARNING]: Collection kubernetes.core does not support Ansible version 2.10.8

PLAY [Linux host setup] *********************************************************************************************************************************************************************************

TASK [helm-busybox : Update repo cache to show additions] ***********************************************************************************************************************************************
Wednesday 10 January 2024  17:00:44 +0200 (0:00:00.009)       0:00:00.009 *****
ok: [kube1]

TASK [helm-busybox : Deploy Busybox] ********************************************************************************************************************************************************************
Wednesday 10 January 2024  17:00:45 +0200 (0:00:01.428)       0:00:01.437 *****
changed: [kube1]

TASK [helm-busybox : debug] *****************************************************************************************************************************************************************************
Wednesday 10 January 2024  17:00:46 +0200 (0:00:00.887)       0:00:02.324 *****
ok: [kube1] =>
  msg: |-
    Release "busybox" does not exist. Installing it now.
    NAME: busybox
    LAST DEPLOYED: Wed Jan 10 17:00:46 2024
    NAMESPACE: test1
    STATUS: deployed
    REVISION: 1
    NOTES:
    Busybox

    namespace test1
    app busybox

PLAY RECAP **********************************************************************************************************************************************************************************************
kube1                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Wednesday 10 January 2024  17:00:46 +0200 (0:00:00.019)       0:00:02.344 *****
===============================================================================
helm-busybox : Update repo cache to show additions ----------------------------------------------------------------------------------------------------------------------------------------------- 1.43s
helm-busybox : Deploy Busybox -------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.89s
helm-busybox : debug ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.02s
```

Pod check from control-plane:

```text
k8s-admin@kube1:~$ kubectl get pods -n test1
NAME                     READY   STATUS    RESTARTS   AGE
busybox-q4cmf            1/1     Running   0          46s
nginx-7975748858-w4zwj   1/1     Running   0          3h5m
```

