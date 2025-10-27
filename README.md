# OpenShift 4 Bare Metal Install - User Provisioned Infrastructure (UPI)

---

## ✨ Architecture Diagram [ Pic ref : ryhanray ]

> *<img width="2062" height="1522" alt="image" src="https://github.com/user-attachments/assets/160c71e6-ba65-4457-a702-690e2bf76260" />*

---

## 📂 Download Software

1. Download CentOS 8 x86\_64 image
2. Log in to [Red Hat OpenShift Cluster Manager](https://console.redhat.com/openshift)
3. Select **Create Cluster** ➔ **Red Hat OpenShift Container Platform** ➔ **Run on Bare Metal**
4. Download:

   * OpenShift Installer for Linux
   * Pull Secret
   * CLI (Linux + Workstation)
   * RHCOS ISO and RAW images

     * `rhcos-X.X.X-x86_64-metal.x86_64.raw.gz`
     * `rhcos-X.X.X-x86_64-live.x86_64.iso` *(for newer versions)*

---

## 🛠️ Prepare the 'Bare Metal' Environment

**Platform**: Ubuntu 22.04 + VMware Workstation

### VM Configurations

* **Control Plane Nodes (1 node example)**

  * Name: `ocp-cp-1`
  * 2 vCPU, 8 GB RAM, 50 GB Disk
  * NIC: OCP Network
  * ISO: RHCOS

* **Worker Nodes (1 nodes)**

  * Names: `ocp-w-1`
  * Specs same as control node

* **Bootstrap Node**

  * Name: `ocp-bootstrap`
  * 2 vCPU, 8 GB RAM, 50 GB Disk
  * ISO: RHCOS

* **Bastion Node (Services)**

  * Name: `bastion`
  * 2 vCPU, 4 GB RAM, 120 GB Disk
  * NIC1: LAN, NIC2: OCP Network
  * ISO: CentOS 8

> After booting all VMs once, shut down all except bastion.

---

## 🛋️ Configure Environmental Services

### 🚀 Move Installer Files to Bastion or use wget to download directly from the url

```bash
scp openshift-install*.tar.gz openshift-client*.tar.gz rhcos*.raw.gz root@<bastion_ip>:/root/
```

### ✈️ SSH into Bastion and Setup

```bash
ssh root@<bastion_ip>
tar -xvf openshift-client-linux.tar.gz
mv oc kubectl /usr/local/bin
kubectl version && oc version
tar -xvf openshift-install-linux.tar.gz
dnf update -y
dnf install git -y
git clone https://github.com/cubensquare/ocp-baremetal-upi-install
```

### ⚖️ Set Static IP (using nmtui)

* Interface: `ens36`
* Address: `192.168.22.1`
* DNS: `127.0.0.1`
* Domain: `ocp.lan`
* Do not use default route

### 🛡️ Setup firewalld Zones

```bash
nmcli connection modify ens33 connection.zone internal
nmcli connection modify ens37 connection.zone external
firewall-cmd --get-active-zones
firewall-cmd --set-default-zone=internal
firewall-cmd --zone=external --add-masquerade --permanent
firewall-cmd --zone=internal --add-masquerade --permanent
firewall-cmd --reload
```

---

## 🔍 DNS Configuration (BIND)

```bash
dnf install bind bind-utils -y
cp ~/ocp-baremetal-upi-install/dns/named.conf /etc/named.conf
cp -R ~/ocp-baremetal-upi-install/dns/zones /etc/named/
firewall-cmd --add-port=53/udp --zone=internal --permanent
firewall-cmd --add-port=53/tcp --zone=internal --permanent
firewall-cmd --reload
systemctl enable --now named
```

Edit LAN NIC DNS to `127.0.0.1` and test with:

nmtcui edit ens33 
## set the DNS server to local and add the domain name 
## DNS server : 127.0.0.1
## search domain : ocp.lan
## Enable 'ignore automatically obtained DNS parameter'

```bash
dig ocp.lan
dig -x 192.168.22.200
```

---

## 🌐 DHCP Setup

```bash
dnf install dhcp-server -y
cp ~/ocp4-metal-install/dhcpd.conf /etc/dhcp/dhcpd.conf
firewall-cmd --add-service=dhcp --zone=internal --permanent
firewall-cmd --reload
systemctl enable --now dhcpd
```

---

## 🏠 Apache Web Server Setup

```bash
dnf install httpd -y
sed -i 's/Listen 80/Listen 0.0.0.0:8080/' /etc/httpd/conf/httpd.conf
firewall-cmd --add-port=8080/tcp --zone=internal --permanent
firewall-cmd --reload
systemctl enable --now httpd
curl localhost:8080
```

---

## 🔧 HAProxy Setup

```bash
dnf install haproxy -y
cp ~/ocp4-metal-install/haproxy.cfg /etc/haproxy/haproxy.cfg
firewall-cmd --add-port=6443/tcp --zone=internal --permanent
firewall-cmd --add-port=6443/tcp --zone=external --permanent
firewall-cmd --add-port=22623/tcp --zone=internal --permanent
firewall-cmd --add-service={http,https} --zone=internal --permanent
firewall-cmd --add-port=9000/tcp --zone=external --permanent
firewall-cmd --reload
setsebool -P haproxy_connect_any 1
systemctl enable --now haproxy
```

---

## 📂 NFS for Registry [ Optional if you are using a minimal hardware setup for testing purpose ]

```bash
dnf install nfs-utils -y
mkdir -p /shares/registry
chown -R nobody:nobody /shares/registry
chmod -R 777 /shares/registry
echo "/shares/registry 192.168.22.0/24(rw,sync,root_squash,no_subtree_check,no_wdelay)" > /etc/exports
exportfs -rv
firewall-cmd --zone=internal --add-service={mountd,rpc-bind,nfs} --permanent
firewall-cmd --reload
systemctl enable --now nfs-server rpcbind nfs-mountd
```

---

## 📁 Generate and Host Install Files

```bash
ssh-keygen
mkdir ~/ocp-install
cp ~/ocp4-metal-install/install-config.yaml ~/ocp-install
# Edit pull secret and SSH key
vim ~/ocp-install/install-config.yaml

# Generate manifests
openshift-install create manifests --dir ~/ocp-install
# (Optional) Disable master schedulability
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' ~/ocp-install/manifests/cluster-scheduler-02-config.yml

# Generate ignition
openshift-install create ignition-configs --dir ~/ocp-install
mkdir /var/www/html/ocp4
cp -R ~/ocp-install/* /var/www/html/ocp4
mv ~/rhcos*.raw.gz /var/www/html/ocp4/rhcos
chcon -R -t httpd_sys_content_t /var/www/html/ocp4/
chown -R apache: /var/www/html/ocp4/
chmod 755 /var/www/html/ocp4/
curl localhost:8080/ocp4/
```

---

## 🌌 Deploy OpenShift (RHCOS Boot Parameters - run this once the bootstrap vm is up )

> Use following kernel parameters or command after booting via ISO:

```bash
sudo -i
coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.22.1:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.22.1:8080/ocp4/<bootstrap|master|worker>.ign

# Or use this from live environment:
sudo coreos-installer install /dev/sda -u http://192.168.22.1:8080/ocp4/rhcos -I http://192.168.22.1:8080/ocp4/<bootstrap|master|worker>.ign --insecure --insecure-ignition
```

---

## ⏳ Monitor Bootstrap Progress on bastion node

```bash
openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
```

---

## ❌ Remove Bootstrap Node

```bash
vim /etc/haproxy/haproxy.cfg  # Remove bootstrap entries
systemctl reload haproxy
# Remove bootstrap VM
```

---

## 🕗 Complete Installation

```bash
openshift-install --dir ~/ocp-install wait-for install-complete
```

---

## 👩‍💻 Join Worker Nodes

```bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
oc get nodes

# Approve CSRs
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
watch -n5 oc get nodes
```

---

## 🏢 Configure Storage for Registry

```bash
oc edit configs.imageregistry.operator.openshift.io
# Set:
# managementState: Managed
# storage:
#   pvc:
#     claim: ""

oc get pvc -n openshift-image-registry
oc create -f ~/ocp4-metal-install/manifest/registry-pv.yaml
oc get pvc -n openshift-image-registry
```

---

## 🔑 Create First Admin User

```bash
oc apply -f ~/ocp4-metal-install/manifest/oauth-htpasswd.yaml
oc adm policy add-cluster-role-to-user cluster-admin admin
```

---

## 📆 Access OpenShift Console

```bash
oc get co
# On your workstation:
sudo vi /etc/hosts
# Add entry for apps + console + api:
192.168.0.96 bastion api.lab.ocp.lan console-openshift-console.apps.lab.ocp.lan oauth-openshift.apps.lab.ocp.lan
```

Visit `https://console-openshift-console.apps.lab.ocp.lan`
Login with `admin/password` or `kubeadmin` from:

```bash
cat ~/ocp-install/auth/kubeadmin-password
```
