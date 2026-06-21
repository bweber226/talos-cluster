# Talos Cluster Configuration

Managed with [talhelper](https://github.com/budimanjojo/talhelper). Edit `talconfig.yaml` and run `talhelper genconfig` to regenerate machine configs. Never edit files in `clusterconfig/` directly — they are generated and gitignored.

## Cluster Info

| Node | Role | IP | Install Disk | Storage Disk |
|------|------|----|-------------|--------------|
| talos-control-01 | Control Plane | 192.168.0.200 | /dev/nvme0n1 | — |
| talos-worker-01 | Worker | 192.168.0.210 | /dev/sdb | /dev/nvme0n1 (Longhorn) |
| talos-worker-02 | Worker | 192.168.0.211 | /dev/sda | /dev/nvme0n1 (Longhorn) |

- **VIP:** 192.168.0.230
- **Talos Image:** `factory.talos.dev/installer/613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245`
- **Extensions:** siderolabs/iscsi-tools, siderolabs/util-linux-tools (required for Longhorn)

---

## Adding a Worker Node

1. Add the node entry to `talconfig.yaml`:
   ```yaml
   - hostname: talos-worker-03
     ipAddress: 192.168.0.212
     installDisk: /dev/sda   # verify with: talosctl get disks -i -e <dhcp-ip> -n <dhcp-ip>
     controlPlane: false
     networkInterfaces:
       - interface: eno2
         dhcp: false
         addresses:
           - 192.168.0.212/24
         routes:
           - network: 0.0.0.0/0
             gateway: 192.168.0.1
   ```

2. Regenerate configs:
   ```bash
   talhelper genconfig
   ```

3. Boot the new node from the Talos ISO and wait for maintenance mode.

4. Apply the config (use the node's temporary DHCP IP):
   ```bash
   talosctl apply-config \
     --talosconfig=./clusterconfig/talosconfig \
     --nodes=<dhcp-ip> \
     --file=./clusterconfig/home-cluster-talos-worker-03.yaml \
     --insecure
   ```

5. The node will install Talos, reboot, and automatically join the cluster. Verify:
   ```bash
   kubectl get nodes
   ```

---

## Adding a Control Plane Node

> Control plane nodes join the etcd cluster automatically — no additional bootstrap needed after the first node.

1. Add the node entry to `talconfig.yaml` with `controlPlane: true` and include a VIP on the interface:
   ```yaml
   - hostname: talos-control-02
     ipAddress: 192.168.0.201
     installDisk: /dev/nvme0n1
     controlPlane: true
     networkInterfaces:
       - interface: eno2
         dhcp: false
         addresses:
           - 192.168.0.201/24
         routes:
           - network: 0.0.0.0/0
             gateway: 192.168.0.1
         vip:
           ip: 192.168.0.230
   ```

2. Regenerate configs:
   ```bash
   talhelper genconfig
   ```

3. Boot the node from the Talos ISO and apply the config:
   ```bash
   talosctl apply-config \
     --talosconfig=./clusterconfig/talosconfig \
     --nodes=<dhcp-ip> \
     --file=./clusterconfig/home-cluster-talos-control-02.yaml \
     --insecure
   ```

4. The node will join etcd automatically. Verify etcd membership:
   ```bash
   talosctl etcd members --talosconfig=./clusterconfig/talosconfig --nodes=192.168.0.200
   ```

---

## Upgrading Talos OS

Upgrade workers first, then the control plane. Always upgrade one node at a time.

1. Update `talosVersion` in `talconfig.yaml` to the new version.

2. Regenerate configs:
   ```bash
   talhelper genconfig
   ```

3. Get the upgrade commands:
   ```bash
   talhelper gencommand upgrade
   ```

4. Run the upgrade commands one at a time, waiting for each node to come back `Ready` before moving to the next:
   ```bash
   kubectl get nodes -w
   ```

> The schematic ID in `talosImageURL` stays the same across upgrades unless you need to add or change extensions.

---

## Upgrading Kubernetes

Kubernetes upgrades only need to be initiated from one node — Talos handles rolling it out across the cluster.

1. Update `kubernetesVersion` in `talconfig.yaml`.

2. Run the upgrade (target the control plane):
   ```bash
   talosctl upgrade-k8s \
     --talosconfig=./clusterconfig/talosconfig \
     --nodes=192.168.0.200
   ```

3. Monitor progress:
   ```bash
   kubectl get nodes
   ```

---

## Useful Day-to-Day Commands

```bash
# Check node health
kubectl get nodes

# Check Talos service status on a node
talosctl --talosconfig=./clusterconfig/talosconfig -n 192.168.0.200 services

# Read logs from a node
talosctl --talosconfig=./clusterconfig/talosconfig -n 192.168.0.210 logs kubelet

# Check disk layout on a node (maintenance mode)
talosctl get disks -i -e <ip> -n <ip>

# View etcd members
talosctl --talosconfig=./clusterconfig/talosconfig -n 192.168.0.200 etcd members

# Regenerate machine configs after talconfig.yaml changes
talhelper genconfig

# Get kubeconfig (if lost or expired)
talosctl kubeconfig \
  --talosconfig=./clusterconfig/talosconfig \
  --nodes=192.168.0.200 \
  --force ~/.kube/config
```
