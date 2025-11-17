# OpenStack Deployment for O2 IMS Testing

![Overall](../assets/O2-IMS-PTI-Development.png)

## Objective
Deploy minimal OpenStack on RHEL 9 for OSC PTI O2 IMS testing and bare metal provisioning via Redfish.

---

## Deployment Progress Tracker

| Task | Status |
|------|--------|
| [Install Centos 9 on controller VM](#11-controller-vm-setup) | ✅ Complete |
| [Install Centos 9 on bare metal compute](#12-compute-node-setup) | ✅ Complete |
| [Configure Podman + compatibility layer](#13-container-runtime) | ✅ Complete |
| [Setup SSH keys and passwordless sudo](#14-access-configuration) | ✅ Complete |
| [Create Python venv and install Kolla](#21-python-environment) | ✅ Complete |
| [Configure inventory and host_vars](#22-inventory-configuration) | ✅ Complete |
| [Configure globals.yml](#23-global-configuration) | ✅ Complete |
| [Download Ironic deploy images](#24-ironic-images) | ✅ Complete |
| [Generate passwords](#31-password-generation) | ✅ Complete |
| [Bootstrap servers](#32-bootstrap) | ✅ Complete |
| [Run prechecks](#33-prechecks) | ✅ Complete |
| [Deploy full stack](#34-deployment) | ✅ Complete |
| [Post-deploy configuration](#35-post-deploy) | ✅ Complete |
| [Install OpenStack CLI](#41-cli-installation) | ✅ Complete |
| [Integrate OSC IMS Module with OpenStack](#41-cli-installation) | N/A |
| [Integrate OSC IMS Module with BMW FOCOM and NFO](#41-cli-installation) | N/A |
---

## 1. Infrastructure Setup

### 1.1 Controller VM Setup
- **Node**: 192.168.8.51
- **OS**: Centos 9
- **Resources**: 8GB RAM, 40GB disk
- **Status**: ✅ Complete

**Network Configuration**:
- Interface: enp1s0 (192.168.8.51/24)
- External: enp8s0 (for Neutron, no IP)

### 1.2 Compute Node Setup
- **Node**: 192.168.8.35
- **OS**: Centos 9
- **Resources**: Available for hypervisor
- **Status**: ✅ Complete

**Network Configuration**:
- Interface: ens3 (192.168.8.35/24)
- Note: Single interface configuration

### 1.3 Baremetal Node (RT-Workers)

| Node      | Purpose       | Provisioned By   | Redfish Support | CPU | Memory | NIC |
| --        | --            | --               | --  | --  | --     | --  |
| Joule     | K8S Worker-RT | Redfish API      | Yes | Saphire    |        |     |
| Lavoisier | Worker-RT     | Redfish API      | Yes | Saphire   |        |     |
| Newton    | OKD Worker-RT | OpenStack [ToDo] | Yes | Broadwell    |        |     |
| Einstein  | Worker-RT     | Manual           | No    | Haskell    |        |     |


### 1.3 Container Runtime
```bash
# Install Podman compatibility layer
sudo dnf install -y podman-docker
sudo systemctl enable --now podman.socket

# Install Python bindings (system-wide for Ansible)
sudo pip3 install podman
```

### 1.4 Access Configuration
```bash
# SSH keys
ssh-keygen -t rsa
ssh-copy-id root@192.168.8.35

# Passwordless sudo
echo "$USER ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/$USER
```

---

## 2. Kolla-Ansible Installation

### 2.1 Python Environment
```bash
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate
pip install -U pip
pip install 'ansible-core>=2.14,<2.16'
pip install kolla-ansible==18.1.0
```

### 2.2 Inventory Configuration

**File**: `~/multinode`
```ini
[control]
controller ansible_host=192.168.8.51 ansible_connection=local

[network]
controller ansible_connection=local

[compute]
baremetal-01 ansible_host=192.168.8.35 ansible_user=root ansible_ssh_private_key_file=/home/infidel/.ssh/id_rsa

[monitoring]
controller ansible_connection=local

[storage]
controller ansible_connection=local

[deployment]
localhost ansible_connection=local

[baremetal:children]
control
network
compute
monitoring
storage
```
**File**: `~/host_vars/controller.yml`
```yaml
network_interface: "enp1s0"
api_interface: "enp1s0"
neutron_external_interface: "enp8s0"
```

**File**: `~/host_vars/baremetal-01.yml`
```yaml
network_interface: "ens3"
api_interface: "ens3"
tunnel_interface: "ens3"
neutron_external_interface: "ens3"
```

### 2.3 Global Configuration

**File**: `/etc/kolla/globals.yml`
```yaml
kolla_base_distro: "rocky"
kolla_internal_vip_address: "192.168.8.51"
kolla_container_engine: "podman"
network_interface: "enp1s0"
neutron_external_interface: "enp8s0"
enable_neutron_provider_networks: "yes"

# Disable unnecessary services
enable_cinder: "no"
enable_fluentd: "no"
enable_haproxy: "no"
enable_host_os_checks: false

# O2 IMS Requirements
enable_watcher: "yes"

# Ironic - Redfish virtual media only
enable_ironic: "yes"
enable_ironic_ipxe: "no"
enable_ironic_pxe_uefi: "no"
enable_ironic_inspector: "no"
enable_ironic_neutron_agent: "yes"
ironic_enabled_boot_interfaces: ['redfish-virtual-media']
ironic_enabled_deploy_interfaces: ['direct']
ironic_default_boot_interface: 'redfish-virtual-media'
ironic_default_deploy_interface: 'direct'
```

### 2.4 Ironic Images

**Download IPA (Ironic Python Agent)**:
```bash
sudo mkdir -p /etc/kolla/config/ironic
cd /etc/kolla/config/ironic

sudo wget https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/ipa-centos9-stable-2024.1.kernel \
  -O ironic-agent.kernel

sudo wget https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/ipa-centos9-stable-2024.1.initramfs \
  -O ironic-agent.initramfs
```

---

## 3. OpenStack Deployment

### 3.1 Password Generation
```bash
cd ~
source ~/kolla-venv/bin/activate
kolla-genpwd
```

### 3.2 Bootstrap
```bash
kolla-ansible -i ~/multinode bootstrap-servers
```

**Actions**:
- Install Docker/Podman dependencies
- Configure system requirements
- Prepare nodes for deployment

### 3.3 Prechecks
```bash
kolla-ansible -i ~/multinode prechecks
```

**Validates**:
- Network connectivity
- System requirements
- Configuration correctness

### 3.4 Deployment
```bash
kolla-ansible -i ~/multinode deploy
```

**Duration**: 15-30 minutes
**Actions**: Pull images, deploy containers, configure services

### 3.5 Post-Deploy
```bash
kolla-ansible -i ~/multinode post-deploy
```

**Generates**: `/etc/kolla/admin-openrc.sh` (credentials file)

---

## 4. Service Verification

### 4.1 CLI Installation
```bash
source ~/kolla-venv/bin/activate
pip install python-openstackclient python-watcherclient
source /etc/kolla/admin-openrc.sh
```

### 4.2 Core Services
```bash
# Verify all services
openstack service list

# Check compute services
openstack compute service list

# Verify hypervisor
openstack hypervisor list
```

**Expected Output**:
| Service | Type | Status |
|---------|------|--------|
| keystone | identity | ✅ |
| nova | compute | ✅ |
| neutron | network | ✅ |
| glance | image | ✅ |
| placement | placement | ✅ |
| heat | orchestration | ✅ |

### 4.3 Watcher Verification
```bash
# Verify Watcher services
openstack optimize service list

# List available strategies
openstack optimize strategy list

# Check running containers
sudo podman ps | grep watcher
```

**Expected Containers**:
- `watcher_api`
- `watcher_decision_engine`
- `watcher_applier`

### 4.4 Ironic Verification
```bash
# List bare metal drivers
openstack baremetal driver list

# Check Ironic containers
sudo podman ps | grep ironic
```

**Status**: ⚠️ Partial - API/Conductor running, dnsmasq volume issue

**Expected Containers**:
- `ironic_api` ✅
- `ironic_conductor` ✅
- `ironic_dnsmasq` ❌ (volume error)

### 4.5 VM Testing
```bash
# Download test image
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

# Upload image
openstack image create "cirros" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public

# Create network
openstack network create --share --provider-network-type vxlan demo-net
openstack subnet create --network demo-net \
  --subnet-range 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  --allocation-pool start=10.0.0.100,end=10.0.0.200 \
  demo-subnet

# Create flavor
openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny

# Launch VM
openstack server create --image cirros --flavor m1.tiny \
  --network demo-net test-vm

# Verify
openstack server list
```

---

## 5. StarlingX Installation


### Post Installation

![](../assets/timeline.png)

- Machine: Galileo
- ISO: StarlingX.iso
-

#### Post Installation

1. Boot into starlingx.iso by any means (USB Mount, PXE, or Virtual-Media)
2. Choose AIO cli installation (automatic)

### Controller Bootstrap

Populate cluster setup

```yaml
system_mode: simplex
dns_servers:
  - 8.8.8.8
  - 8.8.4.4
external_oam_subnet: 192.168.8.0/24
external_oam_gateway_address: 192.168.8.9
external_oam_floating_address: 192.168.8.35
external_oam_node_0_address: 192.168.8.11
admin_username: admin
admin_password: Bmwlabece@123
ansible_become_pass: Bmwlabece@123
```
### Controller Setup

#### OAM Interface Configuration

```bash
[sysadmin@localhost ~(keystone_admin)]$ system host-if-modify controller-0 enp5s0f0 -c platform
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| ifname           | enp5s0f0                             |
| iftype           | ethernet                             |
| ports            | ['enp5s0f0']                         |
| imac             | 18:31:bf:7d:ce:8e                    |
| imtu             | 1500                                 |
| ifclass          | platform                             |
| ptp_role         | none                                 |
| aemode           | None                                 |
| schedpolicy      | None                                 |
| txhashpolicy     | None                                 |
| primary_reselect | None                                 |
| uuid             | 8b5c760a-20a6-4551-aba2-969208cd7723 |
| ihost_uuid       | 25b378b0-99de-4a0e-b493-8fc0b419c9a2 |
| vlan_id          | None                                 |
| uses             | []                                   |
| used_by          | []                                   |
| created_at       | 2025-11-06T15:34:08.879847+00:00     |
| updated_at       | 2025-11-06T15:52:42.570230+00:00     |
| sriov_numvfs     | 0                                    |
| sriov_vf_driver  | None                                 |
| max_tx_rate      | None                                 |
| ipv4_mode        | None                                 |
| ipv6_mode        | None                                 |
| accelerated      | [True]                               |
+------------------+--------------------------------------+

[sysadmin@localhost ~(keystone_admin)]$ system interface-network-assign controller-0 enp5s0f0 oam
+--------------+--------------------------------------+
| Property     | Value                                |
+--------------+--------------------------------------+
| hostname     | controller-0                         |
| uuid         | c579fca3-c8b2-4aec-9610-413d1fcd84ac |
| ifname       | enp5s0f0                             |
| network_name | oam                                  |
+--------------+--------------------------------------+
```

#### NTP Setup
```bash
[sysadmin@localhost ~(keystone_admin)]$ system ntp-modify ntpservers=0.pool.ntp.org,1.pool.ntp.org
+--------------+--------------------------------------+
| Property     | Value                                |
+--------------+--------------------------------------+
| uuid         | d4504a82-37b4-462c-a1a5-1c348966d700 |
| ntpservers   | 0.pool.ntp.org,1.pool.ntp.org        |
| isystem_uuid | 26b80906-99ec-48fd-b8cc-768781aa0964 |
| created_at   | 2025-11-06T15:33:05.113521+00:00     |
| updated_at   | None                                 |
+--------------+--------------------------------------+
```

#### Host Filesystems
```bash
# Instances filesystem for VM ephemeral storage
[sysadmin@controller-0 ~(keystone_admin)]$ system host-fs-add controller-0 instances=100
+----------------+--------------------------------------+
| Property       | Value                                |
+----------------+--------------------------------------+
| uuid           | 8ecab35d-acc7-4db9-ac7b-6aecdb2b5c9d |
| name           | instances                            |
| size           | 100                                  |
| logical_volume | instances-lv                         |
| state          | Creating (on unlock)                 |
| capabilities   | {'functions': []}                    |
| created_at     | 2025-11-06T16:49:59.675244+00:00     |
| updated_at     | None                                 |
+----------------+--------------------------------------+

# Filesystem allocation summary (1394GB total disk)
[sysadmin@controller-0 ~(keystone_admin)]$ system host-fs-list controller-0
+--------------------------------------+-----------+------+--------------+----------------------+----------------------------+
| UUID                                 | FS Name   | Size | Logical      | State                | Capabilities               |
|                                      |           | in   | Volume       |                      |                            |
|                                      |           | GiB  |              |                      |                            |
+--------------------------------------+-----------+------+--------------+----------------------+----------------------------+
| 55bf0031-a10a-4022-8a57-195b156588b1 | backup    | 25   | backup-lv    | In-Use               | {'functions': []}          |
| 9dc5d124-986b-4a4c-bc2c-88d94a0790b7 | ceph      | 20   | ceph-lv      | Ready                | {'functions': ['monitor']} |
| 5a197360-3f1e-4a0f-a094-24cef6c0e841 | docker    | 60   | docker-lv    | In-Use               | {'functions': []}          |
| 8ecab35d-acc7-4db9-ac7b-6aecdb2b5c9d | instances | 100  | instances-lv | Creating (on unlock) | {'functions': []}          |
| 16d11eda-18ca-45b0-9a96-dd4e16d0fb22 | kubelet   | 10   | kubelet-lv   | In-Use               | {'functions': []}          |
| 37d022c9-7225-44de-b977-3d805914166c | log       | 8    | log-lv       | In-Use               | {'functions': []}          |
| 88c58f0b-0835-4fab-b58b-6c8846055684 | root      | 20   | root-lv      | In-Use               | {'functions': []}          |
| d1c043fd-844a-4a4c-97b5-5cf03ee09199 | scratch   | 16   | scratch-lv   | In-Use               | {'functions': []}          |
| 8d7d4e47-e011-43a9-84c5-3746835fed17 | var       | 20   | var-lv       | In-Use               | {'functions': []}          |
+--------------------------------------+-----------+------+--------------+----------------------+----------------------------+
# Total allocated: ~279GB, Remaining: ~1115GB available
```

#### Data Network Configuration
```bash
# Available network interfaces
[sysadmin@controller-0 ~(keystone_admin)]$ system host-port-list controller-0
+--------------------------------------+----------+----------+--------------+--------+-----------+-------------+------------------------------------------------+
| uuid                                 | name     | type     | pci address  | device | processor | accelerated | device type                                    |
+--------------------------------------+----------+----------+--------------+--------+-----------+-------------+------------------------------------------------+
| 1ebc0427-9a69-49e4-8cb6-72abb43e0841 | enp3s0f0 | ethernet | 0000:03:00.0 | 0      | 0         | True        | Ethernet Controller X710 for 10GbE SFP+ [1572] |
| a8dd67eb-3242-462c-b215-7a8748593f47 | enp3s0f1 | ethernet | 0000:03:00.1 | 0      | 0         | True        | Ethernet Controller X710 for 10GbE SFP+ [1572] |
| be3d3554-0154-45f6-a58d-8e01e2b9a2a6 | enp5s0f0 | ethernet | 0000:05:00.0 | 0      | 0         | True        | I350 Gigabit Network Connection [1521]         |
| 7befa9f6-0e48-4764-b442-1ffbfb030137 | enp5s0f1 | ethernet | 0000:05:00.1 | 0      | 0         | True        | I350 Gigabit Network Connection [1521]         |
| 50f10d0b-bad5-475b-a6ea-efea3453c2ad | ens1f0   | ethernet | 0000:02:00.0 | 0      | 0         | True        | 82580 Gigabit Network Connection [1516]        |
| f1b98994-538d-4d33-8025-f0fb05eedaf8 | ens1f1   | ethernet | 0000:02:00.1 | 0      | 0         | True        | 82580 Gigabit Network Connection [1516]        |
+--------------------------------------+----------+----------+--------------+--------+-----------+-------------+------------------------------------------------+

# Configure data interface (enp5s0f0 reused for data after platform setup)
[sysadmin@controller-0 ~(keystone_admin)]$ system host-if-modify -m 1500 -n data0 -c data controller-0 8b5c760a-20a6-4551-aba2-969208cd7723
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| ifname           | data0                                |
| iftype           | ethernet                             |
| ports            | ['enp5s0f0']                         |
| imac             | 18:31:bf:7d:ce:8e                    |
| imtu             | 1500                                 |
| ifclass          | data                                 |
| ptp_role         | none                                 |
| aemode           | None                                 |
| schedpolicy      | None                                 |
| txhashpolicy     | None                                 |
| primary_reselect | None                                 |
| uuid             | 8b5c760a-20a6-4551-aba2-969208cd7723 |
| ihost_uuid       | 25b378b0-99de-4a0e-b493-8fc0b419c9a2 |
| vlan_id          | None                                 |
| uses             | []                                   |
| used_by          | []                                   |
| created_at       | 2025-11-06T15:34:08.879847+00:00     |
| updated_at       | 2025-11-06T16:52:07.366991+00:00     |
| sriov_numvfs     | 0                                    |
| sriov_vf_driver  | None                                 |
| max_tx_rate      | None                                 |
| ipv4_mode        | static                               |
| ipv6_mode        | disabled                             |
| accelerated      | [True]                               |
+------------------+--------------------------------------+

# Create datanetwork
[sysadmin@controller-0 ~(keystone_admin)]$ system datanetwork-add datanet0 vlan
+--------------+--------------------------------------+
| Property     | Value                                |
+--------------+--------------------------------------+
| id           | 1                                    |
| uuid         | 62599454-da9f-4021-b059-52ad713e9d7a |
| name         | datanet0                             |
| network_type | vlan                                 |
| mtu          | 1500                                 |
| description  | None                                 |
+--------------+--------------------------------------+

# Assign datanetwork to interface
[sysadmin@controller-0 ~(keystone_admin)]$ system interface-datanetwork-assign controller-0 8b5c760a-20a6-4551-aba2-969208cd7723 datanet0
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| hostname         | controller-0                         |
| uuid             | 2cb3e0bf-d3c4-4f93-947a-f1d1f9a132ae |
| ifname           | data0                                |
| datanetwork_name | datanet0                             |
+------------------+--------------------------------------+
```

#### CPU Configuration for OpenStack
```bash
# Assign platform cores (6 cores on processor 0)
[sysadmin@controller-0 ~(keystone_admin)]$ system host-cpu-modify -f platform -p0 6 controller-0
# Output: Cores 0-5 (and their HT siblings) assigned to Platform

# Assign vSwitch cores (2 cores required for OVS-DPDK)
[sysadmin@controller-0 ~(keystone_admin)]$ system host-cpu-modify -f vswitch -p0 2 controller-0
# Output: Cores 6-7 (and their HT siblings) assigned to vSwitch
# Remaining cores (8-13 on both processors) assigned to Application (VMs)
```

#### Storage Backend
```bash
# Rook-Ceph already configured
[sysadmin@controller-0 ~(keystone_admin)]$ system storage-backend-list
+--------------------------------------+-----------------+-----------+----------------------+----------+----------------+---------------------------------------------+
| uuid                                 | name            | backend   | state                | task     | services       | capabilities                                |
+--------------------------------------+-----------------+-----------+----------------------+----------+----------------+---------------------------------------------+
| 76763271-d927-4b39-ad82-dd41bfb9a2e6 | ceph-rook-store | ceph-rook | configuring-with-app | uploaded | block,         | deployment_model: controller replication: 1 |
|                                      |                 |           |                      |          | filesystem     | min_replication: 1                          |
|                                      |                 |           |                      |          |                |                                             |
+--------------------------------------+-----------------+-----------+----------------------+----------+----------------+---------------------------------------------+
```

#### Applications Status
```bash
[sysadmin@controller-0 ~(keystone_admin)]$ system application-list
+--------------------------+-----------+-------------------------------------------+------------------+----------+-----------+
| application              | version   | manifest name                             | manifest file    | status   | progress  |
+--------------------------+-----------+-------------------------------------------+------------------+----------+-----------+
| cert-manager             | 24.09-79  | cert-manager-fluxcd-manifests             | fluxcd-manifests | applied  | completed |
| dell-storage             | 24.09-26  | dell-storage-fluxcd-manifests             | fluxcd-manifests | uploaded | completed |
| nginx-ingress-controller | 24.09-66  | nginx-ingress-controller-fluxcd-manifests | fluxcd-manifests | applied  | completed |
| oidc-auth-apps           | 24.09-63  | oidc-auth-apps-fluxcd-manifests           | fluxcd-manifests | uploaded | completed |
| platform-integ-apps      | 24.09-144 | platform-integ-apps-fluxcd-manifests      | fluxcd-manifests | uploaded | completed |
| rook-ceph                | 24.09-71  | rook-ceph-fluxcd-manifests                | fluxcd-manifests | uploaded | completed |
+--------------------------+-----------+-------------------------------------------+------------------+----------+-----------+
```

#### System Information
```bash
[sysadmin@controller-0 ~(keystone_admin)]$ cat /etc/build.info
SW_VERSION="24.09"
BUILD_TARGET="Host Installer"
BUILD_TYPE="Formal"
BUILD_ID="20250124T210100Z"
SRC_BUILD_ID="12"

JOB="STX_10.0_build_debian"
BUILD_BY="jenkins"
BUILD_NUMBER="13"
BUILD_HOST="yow2-wrcp2-lx"
BUILD_DATE="2025-01-24 21:01:00 +0000"
```

#### Disk Layout
```bash
[sysadmin@controller-0 ~(keystone_admin)]$ system host-disk-list controller-0
+--------------------------------------+-------------+------------+-------------+----------+---------------+--------------+----------------------------------+-------------------------------------------------+
| uuid                                 | device_node | device_num | device_type | size_gib | available_gib | rpm          | serial_id                        | device_path                                     |
+--------------------------------------+-------------+------------+-------------+----------+---------------+--------------+----------------------------------+-------------------------------------------------+
| 70e317fb-502b-4b6d-9ae4-9eecd6a8ba87 | /dev/sda    | 2048       | HDD         | 1394.375 | 0.0           | Undetermined | 0008077b0dbf92c622588d0030f81201 | /dev/disk/by-path/pci-0000:81:00.0-scsi-0:2:0:0 |
+--------------------------------------+-------------+------------+-------------+----------+---------------+--------------+----------------------------------+-------------------------------------------------+
```

### Next Steps

1. **Unlock controller-0** to apply all configurations
2. **Apply Rook-Ceph application** to complete storage setup
3. **Add Ceph OSDs** from remaining ~1115GB disk space
4. **Download and install stx-openstack** for VM support
```bash
   wget https://mirror.starlingx.windriver.com/mirror/starlingx/release/10.0.0/debian/openstack/outputs/helm-charts/stx-openstack-24.09-0-debian-stable-latest.tgz
   system application-upload stx-openstack-24.09-0-debian-stable-latest.tgz
   system application-apply stx-openstack
```

**Duplex Config**

```
system_mode: duplex
dns_servers:
  - 8.8.8.8
  - 8.8.4.4
external_oam_subnet: 192.168.8.0/24
external_oam_gateway_address: 192.168.8.9
external_oam_floating_address: 192.168.8.35
external_oam_node_0_address: 192.168.8.36
external_oam_node_1_address: 192.168.8.86

management_subnet: 192.168.8.0/24
management_start_address: 192.168.8.50
management_end_address: 192.168.8.100
management_gateway_address: 192.168.8.9

cluster_host_subnet: 192.168.206.0/24

admin_username: admin
admin_password: Bmwlabece@123
ansible_become_pass: Bmwlabece@123
```

## Stage 6: OKD O-Cloud with O2IMS - gNB Deployment Pipeline

### 6.1 Objective
Replicate Chris Wheeler's OKD O-Cloud demonstration and extend it by deploying OAI gNB workload on real-time optimized node cluster. This work follows the reference implementation from O-RAN SC PTI-RTP project and demonstrates full O2IMS-based cluster lifecycle management.

**Reference Demo:** [OKD as an O-Cloud Platform - Chris Wheeler, October 29, 2025](https://drive.google.com/file/d/1mSCsa1p-5dkUpwuqrq4mUDtdNz5Iv28U/view)

**Extension Goal:** Deploy containerized gNB with real-time kernel on O2IMS-provisioned cluster

### 6.2 Architecture
```
[Jumphost: Newton - 192.168.8.53]
    └─ OSC PTI-RTP
           │
           ├─> [O-Cloud Manager: OKD SNO VM]
           │      ├─ OKD 4.19
           │      ├─ Stolostron (ACM)
           │      └─ oran-o2ims operator
           │
           └─> [Node Cluster: Galileo - 18:31:bf:7d:ce:8e]
                  ├─ OKD 4.19 SNO-DU
                  ├─ SR-IOV operator
                  ├─ Real-time kernel
                  └─ OAI gNB workload
```

**Hardware:**
- Newton: Xeon E5-2695 v4, 557GB + 836GB storage, MAC: ac:1f:6b:40:04:b6
- Galileo: TBD specs, MAC: 18:31:bf:7d:ce:8e

### 6.3 Deployment Tracker

| Step   | Task                                                                                | Status      | Remarks                                    |
| ------ | ------                                                                              | --------    | ---------                                  |
| 1      | [Jumphost Infrastructure Setup](#64-step-1-jumphost-infrastructure-setup)           | Finished     | Service configuration pending              |
| 2      | [O-Cloud Manager Cluster Deployment](#65-step-2-o-cloud-manager-cluster-deployment) | Finished     | Need clean deployment with fixed templates |
| 3      | [O2IMS Operator Installation](#66-step-3-o2ims-operator-installation)               | Complete    | -                                          |
| 4      | [Cluster Template Preparation](#67-step-4-cluster-template-preparation)             | Not Started | Waiting for O-Cloud Manager completion     |
| 5      | [Node Cluster Provisioning](#68-step-5-node-cluster-provisioning-galileo)           | Not Started | Galileo hardware not configured            |
| 6      | [Node Cluster Configuration](#69-step-6-node-cluster-configuration)                 | Not Started | Depends on Step 5                          |
| 7      | [gNB Workload Deployment](#610-step-7-gnb-workload-deployment)                      | Not Started | Depends on Step 6                          |

---

### 6.4 Step 1: Jumphost Infrastructure Setup

The jumphost provides essential services for cluster deployment: artifact hosting, network services, and time synchronization.

#### 1.1 Base OS Installation - Complete

Installed CentOS Stream 9 via PXE boot with kickstart automation. Key configuration points:

- Used official mirror instead of local to avoid XFS module issues
- UEFI boot mode with explicit EFI partition
- Single disk selection to prevent autopart confusion
- LVM for root filesystem with 22GB swap

**Kickstart partition scheme:**
```bash
ignoredisk --only-use=sda
clearpart --all --initlabel --drives=sda
part /boot/efi --fstype=efi --size=600 --ondisk=sda
part /boot --fstype=xfs --size=1024 --ondisk=sda
part pv.01 --fstype=lvmpv --ondisk=sda --grow
volgroup cs_rhel9-00 pv.01
logvol / --fstype=xfs --name=root --vgname=cs_rhel9-00 --size=1 --grow
logvol swap --fstype=swap --name=swap --vgname=cs_rhel9-00 --size=22400
```

**Issues encountered:**
- XFS kernel module missing from local mirror - switched to official mirror
- GRUB not installed automatically - manually installed from rescue mode
- Boot priority set incorrectly - changed firmware to boot from HDD first

#### 1.2 httpd Configuration - Pending

Will serve OKD installation artifacts, container images, and ignition files.

**Required setup:**
```bash
dnf install -y httpd
systemctl enable --now httpd
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
mkdir -p /var/www/html/okd/{images,ignition}
```

#### 1.3 dnsmasq Configuration - Pending

Provides DNS resolution and DHCP for cluster nodes.

**Network requirements:**
- DNS zone: lab.local
- DHCP range: 192.168.8.100-192.168.8.200
- Static reservations for O-Cloud Manager and Galileo
- PTR records for reverse DNS

#### 1.4 chronyd Configuration - Pending

NTP server for cluster time synchronization. Critical for PTP-dependent workloads like gNB.

#### 1.5 libvirt/KVM Setup - Pending

Virtualization platform for O-Cloud Manager VM.
```bash
dnf install -y libvirt qemu-kvm virt-install
systemctl enable --now libvirtd
```

---

### 6.5 Step 2: O-Cloud Manager Cluster Deployment

Deploy single-node OKD cluster that hosts the O2IMS operator and Stolostron components for managing downstream node clusters.

#### 2.1 Repository Setup - Complete
```bash
git clone https://github.com/o-ran-sc/pti-rtp.git -b l-release
cd pti-rtp/okd
```

The l-release branch contains automation for OKD 4.19 deployment with integrated O2IMS support.

#### 2.2 Inventory Configuration - Partial

Created host variables but templates need fixes before deployment.

**File: `inventory/host_vars/master-0-vm/vars.yml`**
```yaml
role: master
mac_addresses:
  ens3: "52:54:00:01:23:45"
network_config:
  interfaces:
    - name: ens3
      type: ethernet
      state: up
      ipv4:
        enabled: true
        dhcp: false
        address:
          - ip: 192.168.8.210
            prefix-length: 24
  routes:
    config:
      - destination: 0.0.0.0/0
        next-hop-address: 192.168.8.1
        next-hop-interface: ens3
  dns-resolver:
    config:
      server:
        - 192.168.8.1
        - 8.8.8.8
```

#### 2.3 Template Fixes - Complete

**Issue:** Templates only applied network config for baremetal infrastructure, ignoring VM deployments.

**Fix 1: agent-config.yaml.j2**

Changed from checking infrastructure type to checking if network_config exists:
```jinja2
# roles/ocloud_platform_okd/templates/agent-config.yaml.j2
# Old: {% if ocloud_infra == 'baremetal' %}
# New:
{% if hostvars[hostname]['network_config'] is defined %}
  - role: {{ hostvars[hostname]['role'] }}
    hostname: {{ hostname }}
    {% if hostvars[hostname]['mac_addresses'] is defined %}
    interfaces:
    - name: {{ hostvars[hostname]['network_config']['interfaces'][0]['name'] }}
      macAddress: {{ hostvars[hostname]['mac_addresses']['ens3'] }}
    {% endif %}
    networkConfig:
      {{ hostvars[hostname]['network_config'] | to_nice_yaml | indent(6) }}
{% endif %}
```

**Fix 2: virt.xml.j2**

Added logic to use MAC address from host_vars:
```xml
<!-- roles/ocloud_infra_vm/templates/virt.xml.j2 -->
<interface type='network'>
{% for ocloud_host in groups['ocloud'] %}
{% if ocloud_host == inventory_hostname %}
  {% if hostvars[ocloud_host].get('mac_addresses') and 'ens3' in hostvars[ocloud_host]['mac_addresses'] %}
      <mac address='{{ hostvars[ocloud_host]["mac_addresses"]["ens3"] }}'/>
  {% else %}
      <mac address='{{ ocloud_net_mac_prefix }}:{{ loop.index + 10 }}'/>
  {% endif %}
{% endif %}
{% endfor %}
      <source network='{{ ocloud_net_name }}'/>
      <model type='virtio'/>
</interface>
```

**How it works:**

OpenShift agent-based installer embeds network configuration in the boot ISO. When the VM boots, the agent inside CoreOS reads the embedded config and uses nmstate to match the configuration to the correct host by MAC address. The agent then applies the static IP configuration using NetworkManager inside the guest OS. The hypervisor only needs to assign the correct MAC address to the VM interface.

#### 2.4 Deployment - Pending

After template validation, run:
```bash
ansible-playbook -i inventory playbooks/ocloud.yml
```

This will:
1. Create libvirt network and storage
2. Generate OKD agent ISO with network config
3. Create VM with specified MAC address
4. Boot VM from ISO
5. Wait for cluster installation
6. Install Stolostron components
7. Deploy O2IMS operator

#### 2.5 Stolostron Installation - Pending

The playbook includes role `ocloud_platform_stolostron` which deploys:
- multicluster-engine: Cluster lifecycle management
- multicluster-observability: Monitoring across clusters
- cluster-group-upgrades: Fleet update orchestration
- siteconfig: Integration with Assisted Installer

---

### 6.6 Step 3: O2IMS Operator Installation

Deploy the ORAN O2 Infrastructure Management Services operator that provides the O2 API for cluster provisioning.

#### 3.1 Problem: API Group Mismatch - Resolved

**Discovery:**

Pre-built operator images (4.18.0 through 4.21.0) were compiled with the old API group `ocloud.openshift.io/v1alpha1`, but the CRDs in the osc-l-release branch had migrated to `o2ims.oran.openshift.io/v1alpha1`.

**Error message:**
```
{"time":"2025-11-17T15:13:20Z","level":"ERROR","msg":"Failed to create default inventory CR","error":"failed to create default inventory CR: no matches for kind \"Inventory\" in version \"ocloud.openshift.io/v1alpha1\""}
```

**Verification:**
```bash
# CRDs show new API group
oc get crd inventories.o2ims.oran.openshift.io -o yaml | grep "group:"
  group: o2ims.oran.openshift.io

# But container was looking for old group
oc logs -n oran-o2ims deployment/oran-o2ims-controller-manager
# Shows: no matches for kind "Inventory" in version "ocloud.openshift.io/v1alpha1"
```

#### 3.2 Solution: Build from Source - Complete
```bash
cd /root/ocloud.running/git/oran-o2ims
git checkout origin/osc-l-release

# Install CRDs
make install

# Build container image
make docker-build IMG=localhost/oran-o2ims:osc-l-release

# Push to registry (or use local)
make docker-push IMG=<registry>/oran-o2ims:osc-l-release

# Deploy with custom image
make deploy IMG=<registry>/oran-o2ims:osc-l-release
```

**Build notes:**
- Go 1.23+ required
- `setup-envtest` downloads take ~10 minutes
- Builds controller binary and container image

#### 3.3 Deployment Verification - Complete

**All pods running:**
```bash
oc get pods -n oran-o2ims
NAME                                            READY   STATUS    RESTARTS   AGE
alarms-server-847f9c5fc9-gs69s                  1/1     Running   0          58s
artifacts-server-5f4bf75fb5-pv8gg               1/1     Running   0          58s
cluster-server-777fd9855d-lvwrn                 1/1     Running   0          59s
oran-o2ims-controller-manager-d657dbd9c-hvtjd   1/1     Running   0          78s
postgres-server-5bbbf8bb75-krvsh                1/1     Running   0          59s
provisioning-server-5f45f9f8cc-46xqp            1/1     Running   0          58s
resource-server-7b4756c4b5-ksldb                1/1     Running   0          59s
```

**Services exposed:**
- alarms-server: Infrastructure monitoring alerts
- artifacts-server: Container images and artifacts
- cluster-server: Cluster lifecycle management
- postgres-server: Backend database
- provisioning-server: Provisioning requests
- resource-server: Infrastructure inventory

**Routes created:**
```
o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureProvisioning
o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureCluster
o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureInventory
o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureMonitoring
o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureArtifacts
```

#### 3.4 API Testing - Complete

**Create test client:**
```bash
oc apply -f https://raw.githubusercontent.com/openshift-kni/oran-o2ims/refs/heads/osc-k-release/config/testing/client-service-account-rbac.yaml
TOKEN=$(oc create token -n oran-o2ims test-client)
```

**Add hostname to /etc/hosts:**
```bash
echo "<cluster-ip> o2ims.apps.ocloud-vm-bmw.lab.local" >> /etc/hosts
```

**Test infrastructure inventory endpoint:**
```bash
curl -H "Authorization: Bearer $TOKEN" -k https://o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureInventory/v1 | jq
```

**Response:**
```json
{
  "description": "OpenShift O-Cloud Manager",
  "extensions": {},
  "globalcloudId": "00000000-0000-0000-0000-000000000000",
  "name": "OpenShift O-Cloud Manager",
  "oCloudId": "369d043e-b42d-4afa-ba59-ddd807d74275",
  "serviceUri": "https://o2ims.apps.ocloud-vm-bmw.lab.local"
}
```

The API responds correctly with the O-Cloud ID and service URI. Operator is fully functional.

---

### 6.7 Step 4: Cluster Template Preparation

Create the manifests that define how downstream node clusters should be deployed. These are consumed by the O2IMS provisioning server.

#### 4.1 ClusterImageSet

Defines the OKD release image to deploy.

**File: `clusterimagesets/4.19.0-okd-scos.19.yaml`**
```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: 4.19.0-okd-scos.19
spec:
  releaseImage: quay.io/okd/scos-release:4.19.0-okd-scos.19
```

**Deployment:**
```bash
oc apply -f clusterimagesets/4.19.0-okd-scos.19.yaml
```

#### 4.2 ClusterTemplate - Not Started

Defines the topology and configuration for SNO-DU clusters.

**File: `clustertemplates/okd-4.19/sno-du/sno-du-okd-v4-19.yaml`**

Key sections:
- Template name and version
- Release image reference
- ClusterInstance defaults
- PolicyTemplate defaults
- Template parameter schema

Parameters include:
- `nodeClusterName`: Unique cluster identifier
- `oCloudSiteId`: Physical site location
- `policyTemplateParameters`: SR-IOV config, CPU tuning
- `clusterInstanceParameters`: Network, storage, node specs

**Reference:** See Chris' demo artifacts in pti-rtp repo under `tece-osc/manifests/`

#### 4.3 PolicyGenerator - Not Started

Defines operator subscriptions and configurations to apply post-deployment.

**File: `policytemplates/okd-4.19/sno-du/sno-du-v1.yaml`**

Deploys:
- OKDerators CatalogSource
- SR-IOV Network Operator subscription
- Performance Profile for CPU isolation
- Machine Config for real-time kernel
- Network policies for workload isolation

Uses templating to inject parameters from ProvisioningRequest.

#### 4.4 ProvisioningRequest Template - Not Started

JSON payload submitted to O2IMS API to trigger cluster deployment.

**File: `provisioningrequests/sno-du-galileo.json`**
```json
{
  "provisioningRequestId": "uuid-here",
  "name": "sno-du-galileo",
  "description": "SNO cluster for gNB workload on Galileo",
  "templateName": "sno-du",
  "templateVersion": "okd-v4-19",
  "templateParameters": {
    "nodeClusterName": "sno-du-galileo",
    "oCloudSiteId": "lab-local",
    "policyTemplateParameters": {
      "sriov-network-pfNames-1": "[\"ens1f0\"]",
      "sriov-network-vlan-1": "110",
      "cpu-isolated": "0-1,28-29",
      "cpu-reserved": "2-10"
    },
    "clusterInstanceParameters": {
      "baseDomain": "lab.local",
      "clusterName": "sno-du-galileo",
      "nodes": [
        {
          "hostName": "galileo",
          "bmcAddress": "redfish-virtualmedia://192.168.8.xxx/redfish/v1/Systems/1",
          "bmcCredentialsName": "galileo-bmc-secret",
          "bootMACAddress": "18:31:bf:7d:ce:8e"
        }
      ]
    }
  }
}
```

---

### 6.8 Step 5: Node Cluster Provisioning (Galileo)

Use O2IMS API to provision OKD cluster on Galileo bare metal server.

#### 5.1 Hardware Registration - Not Started

Configure Galileo BMC for remote management and PXE boot.

**Requirements:**
- BMC IP address configured
- Redfish API enabled
- Virtual media support verified
- PXE boot enabled on primary NIC

**Create BMC credentials secret:**
```bash
oc create secret generic galileo-bmc-secret \
  --from-literal=username=admin \
  --from-literal=password=<bmc-password> \
  -n sno-du-galileo
```

#### 5.2 Hardware Discovery - Not Started

O2IMS resource server will discover hardware inventory through BMC.

**Expected discovery:**
- CPU topology and core count
- Memory capacity
- Network interface details (MAC addresses, speed)
- Storage devices
- BMC firmware version

**Verification:**
```bash
curl -H "Authorization: Bearer $TOKEN" -k \
  https://o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureInventory/v1/resourcePools
```

#### 5.3 Apply ClusterTemplate - Not Started
```bash
oc apply -f clustertemplates/okd-4.19/sno-du/sno-du-okd-v4-19.yaml
```

Verify template registered:
```bash
oc get clustertemplates -n sno-du-okd-v4-19
```

#### 5.4 Submit ProvisioningRequest - Not Started
```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @provisioningrequests/sno-du-galileo.json \
  -k https://o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureProvisioning/v1/provisioningRequests
```

#### 5.5 Monitor Provisioning - Not Started

Track deployment progress:
```bash
# Get provisioning request status
curl -H "Authorization: Bearer $TOKEN" -k \
  https://o2ims.apps.ocloud-vm-bmw.lab.local/o2ims-infrastructureProvisioning/v1/provisioningRequests/<uuid>

# Watch cluster deployment
oc get clusterdeployment -n sno-du-galileo -w

# Monitor assisted installer
oc logs -n multicluster-engine deployment/assisted-service -f
```

**Expected flow:**
1. Request validated by provisioning-server
2. ClusterDeployment CR created
3. Assisted Installer generates discovery ISO
4. ISO mounted via BMC virtual media
5. Galileo boots from ISO
6. Agent registers with Assisted Installer
7. Pre-flight checks executed
8. OKD installation begins
9. Cluster API becomes available
10. PolicyGenerator applies configurations
11. Operators deploy and configure

**Duration:** Approximately 45-60 minutes for SNO deployment

#### 5.6 Extract Kubeconfig - Not Started
```bash
# Get admin kubeconfig from secret
oc extract secret/sno-du-galileo-admin-kubeconfig \
  -n sno-du-galileo \
  --to=- > galileo-kubeconfig

# Test access
oc --kubeconfig=galileo-kubeconfig get nodes
```

---

### 6.9 Step 6: Node Cluster Configuration

Configure the deployed cluster for real-time workloads with SR-IOV networking.

#### 6.1 SR-IOV Operator Validation - Not Started

Verify operator deployed by PolicyGenerator:
```bash
oc --kubeconfig=galileo-kubeconfig get pods -n openshift-sriov-network-operator
oc --kubeconfig=galileo-kubeconfig get sriovnetworknodepolicies
```

#### 6.2 SR-IOV Network Configuration - Not Started

Configure physical functions for fronthaul networking:
```bash
oc --kubeconfig=galileo-kubeconfig apply -f - <<EOF
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: fronthaul-policy
  namespace: openshift-sriov-network-operator
spec:
  nodeSelector:
    node-role.kubernetes.io/master: ""
  resourceName: fronthaulnet
  priority: 10
  numVfs: 8
  nicSelector:
    pfNames: ["ens1f0"]
  deviceType: netdevice
EOF
```

Wait for node reboot and VF creation:
```bash
oc --kubeconfig=galileo-kubeconfig get sriovnetworknodestates -n openshift-sriov-network-operator
```

Create SR-IOV network:
```bash
oc --kubeconfig=galileo-kubeconfig apply -f - <<EOF
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: fronthaul-network
  namespace: openshift-sriov-network-operator
spec:
  resourceName: fronthaulnet
  networkNamespace: gnb-namespace
  vlan: 110
  ipam: |
    {
      "type": "host-local",
      "subnet": "10.110.0.0/24"
    }
EOF
```

#### 6.3 CPU Isolation Verification - Not Started

Validate PerformanceProfile applied:
```bash
oc --kubeconfig=galileo-kubeconfig get performanceprofile
oc --kubeconfig=galileo-kubeconfig get machineconfig | grep performance
```

Check CPU isolation on node:
```bash
oc --kubeconfig=galileo-kubeconfig debug node/galileo
chroot /host
cat /sys/devices/system/cpu/isolated
cat /proc/cmdline | grep isolcpus
```

Expected output:
```
isolated: 0-1,28-29
isolcpus=0-1,28-29
```

#### 6.4 Real-time Kernel Validation - Not Started
```bash
oc --kubeconfig=galileo-kubeconfig debug node/galileo
chroot /host
uname -r
# Should show: xxx.xxx.xxx-xxx.rt.xxx
```

#### 6.5 PTP Configuration - Not Started

For fronthaul timing synchronization with O-RU:
```bash
oc --kubeconfig=galileo-kubeconfig apply -f - <<EOF
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: fronthaul-ptp
  namespace: openshift-ptp
spec:
  profile:
  - name: "fronthaul-slave"
    interface: "ens1f0"
    ptp4lOpts: "-s -2"
    phc2sysOpts: "-a -r"
  recommend:
  - profile: "fronthaul-slave"
    priority: 4
    match:
    - nodeLabel: "node-role.kubernetes.io/master"
EOF
```

Verify PTP sync:
```bash
oc --kubeconfig=galileo-kubeconfig logs -n openshift-ptp -l app=linuxptp-daemon
# Look for: "rms xxxx max xxxx" with low values (<100ns for good sync)
```

---

### 6.10 Step 7: gNB Workload Deployment

Deploy OAI gNB container on the configured node cluster.

#### 7.1 Prepare gNB Container - Not Started

Build or pull OAI RAN gNB container image:
```bash
# Option 1: Pull from registry
podman pull docker.io/oaisoftwarealliance/oai-gnb:latest

# Option 2: Build from source
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git
cd openairinterface5g
docker build -t oai-gnb:local -f docker/Dockerfile.gNB.ubuntu .
```

Push to accessible registry:
```bash
podman tag oai-gnb:local <registry>/oai-gnb:v1.0
podman push <registry>/oai-gnb:v1.0
```

#### 7.2 Create gNB Namespace - Not Started
```bash
oc --kubeconfig=galileo-kubeconfig create namespace gnb-namespace
```

#### 7.3 gNB Configuration - Not Started

Create ConfigMap with gNB configuration:
```bash
oc --kubeconfig=galileo-kubeconfig apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: gnb-config
  namespace: gnb-namespace
data:
  gnb.conf: |
    Active_gNBs = ( "gNB-OAI");
    gNBs = ({
      gNB_ID = 0xe00;
      gNB_name = "gNB-OAI";
      plmn_list = ({
        mcc = 208;
        mnc = 95;
        mnc_length = 2;
      });
      nr_cellid = 12345678L;
      tracking_area_code = 1;
      physical_cellId = 0;
      # Fronthaul interface
      local_s_address = "10.110.0.10";
      remote_s_address = "10.110.0.1";  # O-RU address
    });
EOF
```

#### 7.4 Deploy gNB Pod - Not Started
```bash
oc --kubeconfig=galileo-kubeconfig apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gnb-du
  namespace: gnb-namespace
  annotations:
    k8s.v1.cni.cncf.io/networks: fronthaul-network
spec:
  hostNetwork: false
  containers:
  - name: gnb
    image: <registry>/oai-gnb:v1.0
    securityContext:
      privileged: true
      capabilities:
        add:
        - IPC_LOCK
        - SYS_NICE
    resources:
      requests:
        memory: "8Gi"
        cpu: "4"
        openshift.io/fronthaulnet: "1"
      limits:
        memory: "8Gi"
        cpu: "4"
        hugepages-1Gi: "4Gi"
        openshift.io/fronthaulnet: "1"
    volumeMounts:
    - name: config
      mountPath: /opt/oai-gnb/etc
    - name: hugepages
      mountPath: /dev/hugepages
    env:
    - name: TZ
      value: UTC
  nodeSelector:
    node-role.kubernetes.io/master: ""
  volumes:
  - name: config
    configMap:
      name: gnb-config
  - name: hugepages
    emptyDir:
      medium: HugePages
EOF
```

**Key configuration:**
- Privileged mode for hardware access
- SR-IOV VF attachment for fronthaul
- CPU pinning to isolated cores
- Hugepages for DPDK
- Host network disabled to use SR-IOV

#### 7.5 Validation - Not Started

**Check pod status:**
```bash
oc --kubeconfig=galileo-kubeconfig get pod -n gnb-namespace
oc --kubeconfig=galileo-kubeconfig logs -n gnb-namespace gnb-du
```

**Expected log output:**
```
[PHY] RU synced
[MAC] gNB started
[RRC] Waiting for UE connections
```

**Verify SR-IOV attachment:**
```bash
oc --kubeconfig=galileo-kubeconfig exec -n gnb-namespace gnb-du -- ip addr show
# Should show net1 interface with 10.110.0.x address
```

**Check CPU affinity:**
```bash
oc --kubeconfig=galileo-kubeconfig exec -n gnb-namespace gnb-du -- taskset -cp 1
# Should show: pid 1's current affinity list: 0-1,28-29
```

#### 7.6 End-to-End Test - Not Started

**Requirements:**
- O-RU connected on VLAN 110
- 5G Core (AMF/SMF/UPF) accessible
- UE device for testing

**Test procedure:**
1. Verify fronthaul link with O-RU
2. Check L1 synchronization
3. Attach UE to network
4. Validate RRC connection establishment
5. Test data plane throughput

**Performance metrics:**
- Fronthaul latency: 250µs
- CPU utilization on isolated cores
- Packet loss rate
- Throughput: Target 1Gbps+ depending on configuration

---

### 6.11 Known Issues and Workarounds

#### Issue 1: Static IP Not Applied in VMs

**Symptom:** VM gets DHCP address instead of configured static IP

**Root Cause:** Playbook template `agent-config.yaml.j2` only checked for `ocloud_infra == 'baremetal'`, ignoring VM deployments with network_config defined.

**Fix:** Modified template to check if `network_config` exists in host_vars rather than checking infrastructure type. See §6.5.3 for complete fix.

#### Issue 2: VM Ignores Configured MAC Address

**Symptom:** libvirt assigns random MAC instead of using host_vars definition

**Root Cause:** Template `virt.xml.j2` didn't check for mac_addresses in host_vars.

**Fix:** Added conditional logic to use defined MAC when available, fall back to auto-generated otherwise. See §6.5.3 for implementation.

#### Issue 3: O2IMS Operator CrashLoopBackOff

**Symptom:** Controller pod crashes with "no matches for kind Inventory in version ocloud.openshift.io/v1alpha1"

**Root Cause:** Pre-built operator images 4.18-4.21 compiled with old API group, but CRDs migrated to new group in osc-l-release branch.

**Fix:** Built operator from source. See §6.6 for complete procedure.

#### Issue 4: CentOS Stream 9 Kickstart Fails

**Symptom:** Installation fails with "Module xfs not found in kernel"

**Root Cause:** Local mirror ISO extraction incomplete.

**Fix:** Use official CentOS Stream mirror instead of local mirror. See §6.4.1.

---

### 6.12 References

- [O-RAN SC PTI-RTP GitHub](https://github.com/o-ran-sc/pti-rtp)
- [Chris Wheeler Demo Presentation PDF](https://drive.google.com/file/d/1mSCsa1p-5dkUpwuqrq4mUDtdNz5Iv28U/view)
- [Demo Video: O-Cloud Manager Deployment](https://drive.google.com/file/d/1mSCsa1p-5dkUpwuqrq4mUDtdNz5Iv28U/view)
- [Demo Video: Node Cluster Provisioning](https://drive.google.com/file/d/1rffy6rPCgFmZeSBLfuGC23wg1gsGHS79/view)
- [Demo Video: Validation](https://drive.google.com/file/d/15p321jNU2dBrNwjkq_xtTPQi3DDaunHX/view)
- [ORAN O2IMS Operator GitHub](https://github.com/openshift-kni/oran-o2ims)
- [OKD Documentation](https://docs.okd.io/)
- [OpenShift Agent-Based Installer Guide](https://docs.openshift.com/container-platform/4.12/installing/installing_with_agent_based_installer/preparing-to-install-with-agent-based-installer.html)

---

**Last Updated:** November 17, 2025
**Current Phase:** Step 3 Complete - O2IMS API Operational
**Next Milestone:** Complete O-Cloud Manager deployment with validated templates
**Critical Blocker:** Galileo hardware configuration for node cluster provisioning

### Goal

- Replicate Chris' demo for Plugfest
- Contribution Goal: Perform GNB Deployment on O2IMS provisioned Cluster


### Post Installation

![](../assets/timeline.png)

**Master Node**
- Machine: Newton
- MAC: ac:1f:6b:40:04:b6


**Worker Node**
- Machine: Galileo
- MAC: 18:31:bf:7d:ce:8e

---

## 6. O2 IMS Integration

### 6.1 O2 Adapter Installation
**Status**: On Going

**Requirements**:
- O2 IMS service implementation
- O-RAN O2 interface compliance
- Integration with Watcher and Ironic

1. Git Clone project
1. Fetch admin_openrc.sh from openstack

#### 6.1.1 Setup Keycloak Dev mode

1. Crete Local Keycloak Instance
    ```bash
    podman run --name keycloak -p 8080:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin -e KC_HOSTNAME=localhost quay.io/keycloak/keycloak:latest start-dev
    ```
1. Update admin password of keycloak instance
    ```bash
    # From Host
    podman exec -it keycloak /bin/bash
    # From Container
    bash-5.1$ cd /opt/keycloak/bin
    bash-5.1$ ./kcadm.sh config credentials --server http://localhost:8080 --realm master --user admin
    bash-5.1$ ./kcadm.sh update realms/master -s sslRequired=NONE
    ```
1.  Create Keycloak client open keycloak dashboard from `http://localhost:8080`
    1. Goto Clients -> Create Client
    1. asdasldk
    1. asdkjasd

1. Establish communication between O2-IMS and INF (OpenStack)
### 6.2 O2 API Configuration
**Status**: ❌ Not Started

**Configuration Points**:
- O2 IMS endpoint URL
- Authentication credentials
- API version compatibility
- Resource model mapping

### 6.3 Infrastructure Inventory
**Status**: ❌ Not Started

**Tests Required**:
- Query available resources
- Resource pool information
- Deployment capability discovery

### 6.4 Resource Lifecycle
**Status**: ❌ Not Started

**Validation Tests**:
- Bare metal provisioning request
- Resource allocation tracking
- Deprovisioning workflow
- State change notifications

### 6.5 Notification Service
**Status**: ❌ Not Started

**Event Types**:
- Resource state changes
- Alarms and faults
- Capacity warnings
- Lifecycle events

---

## Access Information

### Horizon Dashboard
- **URL**: http://192.168.8.51
- **Username**: admin
- **Password**: `grep keystone_admin_password /etc/kolla/passwords.yml`

### CLI Access
```bash
source ~/kolla-venv/bin/activate
source /etc/kolla/admin-openrc.sh
```

---

## Technical Issues Log

| Issue | Root Cause | Solution | Status | Section |
|-------|------------|----------|--------|---------|
| RHEL not recognized | OS check too strict | `enable_host_os_checks: false` | ✅ Fixed | [2.3](#23-global-configuration) |
| Interface mismatch | Different NIC names | Use host_vars per node | ✅ Fixed | [2.2](#22-inventory-configuration) |
| Podman module missing | Ansible needs bindings | `sudo pip3 install podman` | ✅ Fixed | [1.3](#13-container-runtime) |
| Fluentd mount error | Missing `/var/log/journal` | Disable fluentd | ✅ Fixed | [2.3](#23-global-configuration) |
| Cinder no backend | No storage configured | `enable_cinder: no` | ✅ Fixed | [2.3](#23-global-configuration) |
| Single interface | Only one NIC | Share interface for all traffic | ✅ Fixed | [1.2](#12-compute-node-setup) |
| Ironic dnsmasq error | Volume format issue | Disable inspector/PXE | ⚠️ Workaround | [4.4](#44-ironic-verification) |

---



### StarlingX 10.0 Failed to Ceph install

**Issue**

-

### StarlingX 8.0 Failed to bootstrap due to missing images

**Issue**

- Failed to bootstrap due to missing images (depercated)

```
TASK [common/push-docker-images : debug] ***********************************************************************************************************************************
Wednesday 12 November 2025  11:11:16 +0000 (0:00:00.020)       0:21:00.436 ****
skipping: [localhost]

TASK [common/push-docker-images : Download images and push to local registry] **********************************************************************************************
Wednesday 12 November 2025  11:11:16 +0000 (0:00:00.022)       0:21:00.459 ****
FAILED - RETRYING: Download images and push to local registry (10 retries left).
FAILED - RETRYING: Download images and push to local registry (9 retries left).
FAILED - RETRYING: Download images and push to local registry (8 retries left).
FAILED - RETRYING: Download images and push to local registry (7 retries left).
FAILED - RETRYING: Download images and push to local registry (6 retries left).
FAILED - RETRYING: Download images and push to local registry (5 retries left).
FAILED - RETRYING: Download images and push to local registry (4 retries left).
FAILED - RETRYING: Download images and push to local registry (3 retries left).
FAILED - RETRYING: Download images and push to local registry (2 retries left).
FAILED - RETRYING: Download images and push to local registry (1 retries left).
fatal: [localhost]: FAILED! => changed=true
  attempts: 10
  failed_when_result: true
  msg: non-zero return code
  rc: 1
  stderr: |-
    Traceback (most recent call last):
      File "/tmp/.ansible-sysadmin/tmp/ansible-tmp-1762946472.3770916-85611-204325389315240/download_images.py", line 338, in <module>
        raise Exception("Failed to download images %s" % failed_downloads)
    Exception: Failed to download images ['quay.io/k8scsi/snapshot-controller:v2.0.0-rc2']
  stderr_lines: <omitted>
  stdout: |-
    Image is up to date for sha256:6cab9d1bed1be49c215505c1a438ce0af66eb54b4e95f06e52037fcd36631f3d
    Image is up to date for sha256:1f99cb6da9a82e81081f65acdad10cdca2e5ec4084f91009bdcff31dd6151d48
    Image is up to date for sha256:03fa22539fc1ccdb96fb15098e7a02fff03d0e366ce5d80891eb0a3a8594a0c9
    Image is up to date for sha256:7a53d1e08ef58144850b48d05908b4ef5b611bff99a5a66dbcba7ab9f79433f7
    Image is up to date for sha256:221177c6082a88ea4f6240ab2450d540955ac6f4d5454f0e15751b653ebda165
    Image is up to date for sha256:aebe758cef4cd05b9f8cee39758227714d02f42ef3088023c1e3cd454f927a2b
    Image is up to date for sha256:a4ca41631cc7ac19ce1be3ebf0314ac5f47af7c711f17066006db82ee3b75b03
    Image is up to date for sha256:45f84749206fc21ec0bc8b410c561a2615da1e9192ac6fdf34900436a91c02cf
    Image is up to date for sha256:c595bc026ccfe8ba49cddce7b6479d7bbe98f7999bd29c983a5312ee8fe33323
    Image is up to date for sha256:d4d0783ac01752eab255a7381c936170ff1fc576452f036c5d810e91cf3cea14
    Image is up to date for sha256:0df214aeb2576c81624837fb761fa49ab8433ed62a3144cacd4a947edd3d3a2d
    Image is up to date for sha256:baacffa096025944e4d13e026776e1aaafa3862e74a212780800227526577521
    Image is up to date for sha256:213debde2c3b8c34df33cf3e906168bb7ab94dff4a33ec1c967b1cfd2179fbee
    Image is up to date for sha256:fd3fd9ab134a864eeb7b2c073c0d90192546f597c60416b81fc4166cca47f29a
    Image is up to date for sha256:e88ee986c07782169da593fe29651d9df7ace5ec0ecdb4ca02dc990aa3ccac6b
    Image is up to date for sha256:614615323ea0c7711b1a55922712759ad43b3e65759ff789f614dcbccb20dbd1
    Image is up to date for sha256:7fb3c2364b87e9241db7549bf11d42c129130f44e930d1ce36523fc693186e89
    Image is up to date for sha256:0f8457a4c2ecaceac160805013dc3c61c63a1ff3dee74a473a36249a748e0253
    Image is up to date for sha256:2461b2698dcd5998e3e87a03cedc1bce5b76bfd4ac61ccdef3fc059c64bd8181
    Image is up to date for sha256:c41e9fcadf5a291120de706b7dfa1af598b9f2ed5138b6dcb9f79a68aad0ef4c
    Image is up to date for sha256:b5af743e598496e8ebd7a6eb3fea76a6464041581520d1c2315c95f993287303
    Image is up to date for sha256:d8fece6544a2547e183f84bae63724039c509ba527659f9f65da124f8670948d
    Image is up to date for sha256:2cb486bbac0f5a94783edc046e751ff9600e8aa1786d09255141a1e960e4fc0a
    Image is up to date for sha256:dc6302a19621b6dba29445fcc64b0fa73ac1df36bb02c553ca0e6a54f6ba3395
    Image is up to date for sha256:623ec0d3153915585e02d06b4f76d28339301499a408ca45b6d3e0cbdc38f105
    Image is up to date for sha256:db7725ef729d74e24d51c93f831fa69b22747e67507f6bc2d7c981d16920ff35
    Image is up to date for sha256:36736adc2f4b56924e5e3a46cc8f8eaf9b18c3c6aac310b01c200a3f4a6f400c
    Image is up to date for sha256:4e60e4ce697f12103aeedd200e8abb0352dfd87fd2f64591da2bd923e7824567
    Image k8s.gcr.io/kube-apiserver:v1.24.4 found on local registry
    Image k8s.gcr.io/kube-controller-manager:v1.24.4 found on local registry
    Image k8s.gcr.io/kube-scheduler:v1.24.4 found on local registry
    Image k8s.gcr.io/kube-proxy:v1.24.4 found on local registry
    Image k8s.gcr.io/pause:3.7 found on local registry
    Image k8s.gcr.io/etcd:3.5.3-0 found on local registry
    Image k8s.gcr.io/coredns/coredns:v1.8.6 found on local registry
    Image quay.io/calico/cni:v3.24.0 found on local registry
    Image quay.io/calico/node:v3.24.0 found on local registry
    Image quay.io/calico/kube-controllers:v3.24.0 found on local registry
    Image ghcr.io/k8snetworkplumbingwg/multus-cni:v3.9.2 found on local registry
    Image ghcr.io/k8snetworkplumbingwg/sriov-cni:v2.6.3 found on local registry
    Image ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin:v3.5.1 found on local registry
    Image ghcr.io/helm/tiller:v2.16.9 found on local registry
    Image docker.io/starlingx/armada-image:stx.7.0-v1.0.0 found on local registry
    Image docker.io/starlingx/n3000-opae:stx.8.0-v1.0.2 found on local registry
    Image quay.io/stackanetes/kubernetes-entrypoint:v0.3.1 found on local registry
    Image k8s.gcr.io/pause:3.4.1 found on local registry
    Image k8s.gcr.io/ingress-nginx/controller:v1.1.1 found on local registry
    Image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1 found on local registry
    Image k8s.gcr.io/defaultbackend-amd64:1.5 found on local registry
    Image docker.io/fluxcd/helm-controller:v0.27.0 found on local registry
    Image docker.io/fluxcd/source-controller:v0.32.1 found on local registry
    404 Client Error: Not Found ("manifest unknown: manifest unknown")
    Image quay.io/k8scsi/snapshot-controller:v2.0.0-rc2 not found on local registry, attempt to download...
     Image download failed: quay.io/k8scsi/snapshot-controller:v2.0.0-rc2 500 Server Error: Internal Server Error ("unauthorized: access to the requested resource is not authorized")
    Image quay.io/jetstack/cert-manager-acmesolver:v1.7.1 found on local registry
    Image quay.io/jetstack/cert-manager-cainjector:v1.7.1 found on local registry
    Image quay.io/jetstack/cert-manager-controller:v1.7.1 found on local registry
    Image quay.io/jetstack/cert-manager-webhook:v1.7.1 found on local registry
    Image quay.io/jetstack/cert-manager-ctl:v1.7.1 found on local registry
  stdout_lines: <omitted>

PLAY RECAP *****************************************************************************************************************************************************************
localhost                  : ok=270  changed=107  unreachable=0    failed=1    skipped=329  rescued=0    ignored=0

Wednesday 12 November 2025  11:21:21 +0000 (0:10:04.536)       0:31:04.995 ****
===============================================================================
common/push-docker-images : Download images and push to local registry -------------------------------------------------------------------------------------------- 604.54s
bootstrap/persist-config : Wait for service endpoints reconfiguration to complete --------------------------------------------------------------------------------- 430.13s
bootstrap/apply-manifest : Applying puppet bootstrap manifest ----------------------------------------------------------------------------------------------------- 412.86s
bootstrap/persist-config : Wait for sysinv inventory --------------------------------------------------------------------------------------------------------------- 61.59s
bootstrap/persist-config : Find old registry secrets in Barbican --------------------------------------------------------------------------------------------------- 60.63s
bootstrap/persist-config : Saving config in sysinv database -------------------------------------------------------------------------------------------------------- 47.75s
bootstrap/validate-config : Generate config ini file for python sysinv db population script ------------------------------------------------------------------------ 42.31s
bootstrap/bringup-essential-services : Add loopback interface ------------------------------------------------------------------------------------------------------ 24.05s
bootstrap/persist-config : Restart sysinv-agent and sysinv-api to pick up sysinv.conf update ----------------------------------------------------------------------- 14.33s
common/create-etcd-certs : Generate private key for etcd server and client ------------------------------------------------------------------------------------------ 7.62s
bootstrap/apply-manifest : Generating static config data ------------------------------------------------------------------------------------------------------------ 7.14s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 5.00s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 4.85s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 4.84s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 4.84s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 4.83s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 4.82s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 4.82s
bootstrap/validate-config : Check if the supplied address is a valid domain name or ip address ---------------------------------------------------------------------- 4.81s
bootstrap/persist-config : Append config ini file with Barbican secret uuid ----------------------------------------------------------------------------------------- 3.65s
```



## Network Topology

### Physical Network
- **Subnet**: 192.168.8.0/24
- **Controller**: 192.168.8.51 (enp1s0)
- **Compute**: 192.168.8.35 (ens3)

### Tenant Networks
- **Type**: VXLAN overlays
- **Example Range**: 10.0.0.0/24
- **DHCP**: Neutron-managed (tenant networks only)

### Traffic Flow
```
Management: Controller ←→ Compute (192.168.8.x)
VM Traffic: Tenant VMs ←→ VXLAN tunnel ←→ External
```

---

## Key Commands Reference

### Deployment Operations
```bash
cd ~
source ~/kolla-venv/bin/activate

# Deploy/Redeploy
kolla-ansible -i ~/multinode deploy

# Reconfigure services
kolla-ansible -i ~/multinode reconfigure

# Deploy specific service
kolla-ansible -i ~/multinode deploy --tags watcher
```

### Container Management
```bash
# List containers
sudo podman ps

# Check specific service
sudo podman ps | grep <service>

# View logs
sudo podman logs <container_name>

# Restart container
sudo podman restart <container_name>
```

### OpenStack Operations
```bash
# Load credentials
source /etc/kolla/admin-openrc.sh

# Service status
openstack service list
openstack compute service list
openstack hypervisor list

# Watcher operations
openstack optimize service list
openstack optimize audit create --strategy basic --name test

# Ironic operations
openstack baremetal driver list
openstack baremetal node list
```

---

## Documentation & References

- **Kolla-Ansible**: https://docs.openstack.org/kolla-ansible/latest/
- **O-RAN O2 Specification**: https://orandownloadsweb.azurewebsites.net/specifications
- **Ironic Redfish Driver**: https://docs.openstack.org/ironic/latest/admin/drivers/redfish.html
- **Watcher Documentation**: https://docs.openstack.org/watcher/latest/
- **OpenStack CLI**: https://docs.openstack.org/python-openstackclient/latest/

---

**Environment**: Lab/Test
**OS**: RHEL 9
**Container Engine**: Podman
**Deployment Tool**: Kolla-Ansible 18.1.0
**Last Updated**: 2025-11-03
**Primary Operator**: infidel
**Current State**: OpenStack operational, O2 IMS integration pending
