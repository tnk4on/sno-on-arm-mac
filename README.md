# Single Node OpenShift on ARM Mac
Install Single Node OpenShift on Apple Silicon-based Mac (ARM Mac)

## Prerequisites
- Podman Desktop

## Sample Parameters
**Network**

| Item| Description| Example Value|
|:-|:-|:-|
| Static IP Address 1| IP address of the Mac host| 192.168.1.150|
| Static IP Address 2| IP address of the OpenShift node| 192.168.1.40|
| Network Address| IP network block in CIDR notation| 192.168.1.0/24|
| Default Gateway| Gateway to the public network| 192.168.1.1|

**Domain and Cluster Name**

| Item| Description | Example Value |
|:-|:-|:-|
| Domain|  <base_domain>| home.lab|
| Cluster Name| <cluster_name>| sno|

**Required DNS Records**

| Item| Description | Example Value |
|:-|:-|:-|
| Kubernetes API| api.<cluster_name>. <base_domain>| api.sno.home.lab|
| Kubernetes API（Internal）| api-int.<cluster_name>. <base_domain>| api-int.sno.home.lab|
| Application Ingress traffic| *.apps.<cluster_name>. <base_domain>| *.apps.sno.home.lab|
| OpenShift node| <host_name>. <cluster_name>. <base_domain>| m1-ocp.sno.home.lab|


## Install
### 1. Clone this Git repository
```
git clone https://github.com/tnk4on/sno-on-arm-mac.git
cd sno-on-arm-mac
```

### 2. Get OpenShift Installer and OpenShift CLI
```
export OCP_VERSION=4.13.0-rc.2
export ARCH=aarch64
mkdir src
cd src
curl -LO https://mirror.openshift.com/pub/openshift-v4/$ARCH/clients/ocp/$OCP_VERSION/openshift-install-mac.tar.gz
curl -LO https://mirror.openshift.com/pub/openshift-v4/$ARCH/clients/ocp/$OCP_VERSION/openshift-client-mac.tar.gz
tar zxf openshift-install-mac.tar.gz
tar zxf openshift-client-mac.tar.gz
sudo cp openshift-install oc /usr/local/bin/
cd ..
```

### 3. Get RHCOS ISO

```
mkdir iso
ISO_URL=$(openshift-install coreos print-stream-json | grep location | grep $ARCH | grep iso | cut -d\" -f4)
curl -L $ISO_URL -o iso/rhcos-live.iso
```

### Run DNS server

#### 1. Change Podman machine to root mode

```
podman machine stop
podman machine set --rootful
podman machine start
```

#### 2. Stop `systemd-resolved` on podman machine

```
podman machine ssh sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf
podman machine ssh systemctl restart systemd-resolved
podman machine ssh ss -ltnup
```

#### 3. Modify dnsmasq.conf

```
user=root
port= 53
expand-hosts
log-queries
log-facility=-
local=/home.lab/
domain=home.lab
address=/apps.sno.home.lab/192.168.1.40
address=/api.sno.home.lab/192.168.1.40
address=/api-int.sno.home.lab/192.168.1.40
address=/m1-ocp.sno.home.lab/192.168.1.40
ptr-record=40.1.168.192.in-addr.arpa,m1-ocp.sno.home.lab
rev-server=192.168.1.0/24,127.0.0.1
```

#### 4. Run container

```
podman run -d --rm -p 53:53/udp -v ./dnsmasq.conf:/etc/dnsmasq.conf --name dnsmasq quay.io/crcont/dnsmasq:latest
```

### Change Mac host settings

#### 1. /etc/resolver
```
sudo tee /etc/resolver/sno.home.lab &>/dev/null <<EOF
nameserver 127.0.0.1
EOF
```

#### 2. /etc/hosts
```
sudo tee -a /etc/hosts &>/dev/null <<EOF
192.168.1.40    api.sno.home.lab
EOF
```

### Test boot of RHCOS ISO and confirmation of device name
Create a virtual machine for the OpenShift node in the virtualization software.

[Samples for UTM]

| Item| Sample value|
|:-|:-|
| CPU(Cores)| 8|
| Memory(MB)| 16384|
| Storage(GB)| 100|
| Use Apple Virtualization| On|
| Boot ISO Image| <any path>/iso/rhcos-live.iso|

After starting RHCOS, check the Storage device name and NIC name.
```
sudo fdisk -l
ip a
```

> (Reference)
>
>- NIC: `enp0s1`(UTM), `ens160`(VMware Fusion)
>- Storage Device: `/dev/vda`(UTM), `/dev/nvme0n1`(VMware Fusion NVMe), `/dev/sda`(VMware Fusion SATA)

### update install-config.yaml

Sample `install-config.yaml`
```
apiVersion: v1
baseDomain: home.lab
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
metadata:
  name: sno
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.1.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
bootstrapInPlace:
  installationDisk: /dev/vda
pullSecret: '{"auths":...}' 
sshKey: 'ssh-ed25519 AAAA...'
```

#### 1. Installation disk

```
bootstrapInPlace:
  installationDisk: /dev/vda
```

#### 2. Pull secret

```
pullSecret: '{"auths":...}' 
```

#### 3. SSH key

```
sshKey: 'ssh-ed25519 AAAA...'
```

### Create Ignition file
```
mkdir ocp
cp install-config.yaml ocp
openshift-install --dir=ocp create single-node-ignition-config
```

### Embed Ignition files into RHCOS ISO
#### 1. Register alias command
```
alias coreos-installer='podman run --privileged --rm \
    -v $PWD:/data \
    -w /data quay.io/coreos/coreos-installer:release'
coreos-installer --version
```

#### 2. Embed Ignition files 
```
coreos-installer iso ignition embed -fi ocp/bootstrap-in-place-for-live-iso.ign iso/rhcos-live.iso
```

### Static IP address configuration for OpenShift nodes

#### 1. Set environment variables

```
export IP=192.168.1.40
export GATEWAY=192.168.1.1
export NETMASK=255.255.255.0
export INT=enp0s1
export HOSTNAME=m1-ocp.sno.home.lab
export DNS=192.168.1.150
```

#### 2. Set kernel arguments for ISO

```
coreos-installer iso kargs modify -a "console=ttyS0 rd.neednet=1 ip=${IP}::${GATEWAY}:${NETMASK}:${HOSTNAME}:${INT}:none nameserver=${DNS}" iso/rhcos-live.iso
```

### Boot the RHCOS ISO

During the automatic execution of the installation, the system will reboot twice at the following times.

- 1st: Boot ISO → After completion of writing to disk
- 2nd: Bootstrap start → After Bootstrap completion

### Reset Environment
Delete the installation directory
```
rm -rf ocp
```

Initialize the ISO image
```
coreos-installer iso reset iso/rhcos-live.iso
```