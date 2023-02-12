# k8s
## Build K8S cluster on libvirt
### initially create single host k8s
[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/]

- MEM 2G 
- HDD 10G 
- 2 VCPU

## Get OS cloudimage
```
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
qemu-img info jammy-server-cloudimg-amd64.img
mv jammy-server-cloudimg-amd64.img jammy-server-cloudimg-amd64.qcow2

cat >meta-data <<EOF\nlocal-hostname: instance-1\nEOF
export PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
cat >user-data <<EOF\n#cloud-config\nusers:\n  - name: ubuntu\n    ssh-authorized-keys:\n      - $PUB_KEY\n    sudo: ['ALL=(ALL) NOPASSWD:ALL']\n    groups: sudo\n    shell: /bin/bash\nruncmd:\n  - echo "AllowUsers ubuntu" >> /etc/ssh/sshd_config\n  - restart ssh\nEOF

sudo qemu-img create -f qcow2 -F qcow2 -o backing_file=jammy-server-cloudimg-amd64.qcow2  /var/lib/libvirt/images/k8s.qcow2
sudo qemu-img info  /var/lib/libvirt/images/k8s.qcow2
```
## Resize image to 10G 
```
sudo qemu-img create -f qcow2 -F qcow2 -o backing_file=jammy-server-cloudimg-amd64.qcow2  /var/lib/libvirt/images/k8s.qcow2
```
## Build VM
```
sudo genisoimage  -output /var/lib/libvirt/images/k8s-cidata.iso -volid cidata -joliet -rock user-data meta-data
virt-install --connect qemu:///system --virt-type kvm --name k8s --ram 2048 --vcpus=1 --os-type linux --os-variant ubuntu22.04 --disk path=/var/lib/libvirt/images/k8s.qcow2,format=qcow2 --disk /var/lib/libvirt/images/k8s-cidata.iso,device=cdrom --import --network network=default --noautoconsole
```

## Clone VMs
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
### leave only :
```
network:
    ethernets:
        ens3:
            dhcp4: true
    version: 2

sudo netplan generate 
sudo netplan apply
```
### Clone vms. Here we can create k8s-[a-d] VMs for master node, worker nodes. 
```
virh shutdown k8s
virt-clone --original k8s --name k8s-a --file /var/lib/libvirt/images/k8s-a.qcow2
virt-sysprep -d fork8s-a --hostname k8s-a --enable user-account,ssh-hostkeys,net-hostname,net-hwaddr,machine-id --keep-user-accounts ubuntu --keep-user-accounts amartynov --keep-user-accounts root
```
## Prepare KVM hosts
### Clean all libvirt dhcp data
```
sudo rm /var/lib/libvirt/dnsmasq/virbr0*
sudo rm /var/lib/libvirt/dnsmasq/default.hostsfile
sudo cat /var/lib/libvirt/dnsmasq/default.conf
sudo cat /var/lib/libvirt/dnsmasq/default.addnhosts
```
### Collect MACs
```
sudo virsh dumpxml k8s | grep mac
sudo virsh dumpxml k8s-a| grep mac
sudo virsh dumpxml k8s-b| grep mac
sudo virsh dumpxml k8s-c| grep mac
sudo virsh dumpxml k8s-d| grep mac
```
```
sudo virsh net-edit default
sudo virsh net-destroy default
sudo virsh net-start default

sudo virsh net-dhcp-leases --network default
```
## Ansible inventory
### Easiest is scripts/inventory.sh
```
for i in {10..14}; do echo -n "192.168.122.$i,"; done;
```
### Check avaiability
```
ansible -i "192.168.122.10,192.168.122.11,192.168.122.12,192.168.122.13,192.168.122.14," all -m ping
```
```
ansible -i "192.168.122.10,192.168.122.11,192.168.122.12,192.168.122.13,192.168.122.14," all  -m shell -a "sed -i '/match:/d; /macaddress/d' /etc/netplan/50-cloud-init.yaml"
```