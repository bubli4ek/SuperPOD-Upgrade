# H100 SuperPOD Pre-Upgrade Data Collection

All of the collection commands below are **read-only**. Run them on the **active BCM head node** as root unless a section says otherwise. Everything is written into one timestamped folder, so the customer can attach it to a support case or compare it after the upgrade.

---

## 0. Setup (run first)

```bash
export OUT=/cm/shared/backups/pre-upgrade-$(date +%Y%m%d-%H%M%S)
mkdir -p $OUT/{bcm,nmc,k8s,scheduler,os,drivers,fabric,fw}
cd $OUT
```

---

## 1. Base Command Manager (BCM)

### 1.1 Version and HA state

```bash
cmsh -c "main; versioninfo" > bcm/versioninfo.txt
cm-package-release-info -f cmdaemon > bcm/cmdaemon-version.txt
cmha status > bcm/cmha-status.txt
cmsh -c "partition failover base; get disableautomaticfailover" > bcm/failover-setting.txt
systemctl status cmd --no-pager > bcm/cmd-service.txt
```

### 1.2 Inventory

```bash
cmsh -c "device; list" > bcm/device-list.txt
cmsh -c "device; status" > bcm/device-status.txt
cmsh -c "category; list" > bcm/category-list.txt
cmsh -c "softwareimage; list" > bcm/softwareimage-list.txt
cmsh -c "softwareimage provisioningstatus -r" > bcm/provisioning-status.txt
cmsh -c "network; list" > bcm/network-list.txt
cmsh -c "partition; list" > bcm/partition-list.txt
cmsh -c "kubernetes; list" > bcm/kubernetes-list.txt
cmsh -c "wlm; list" > bcm/wlm-list.txt 2>&1
```

### 1.3 Per-category and per-image detail

```bash
for c in $(cmsh -c "category; list" | awk 'NR>2{print $1}'); do
  cmsh -c "category use $c; show" > bcm/category-$c.txt
done
for i in $(cmsh -c "softwareimage; list" | awk 'NR>2{print $1}'); do
  cmsh -c "softwareimage use $i; show; kernelmodules; list" > bcm/image-$i.txt
done
```

### 1.4 Health

```bash
cmsh -c "device; foreach -v * (latesthealthdata)" > bcm/latest-health.txt
```

### 1.5 Custom file tracking

Collect the `filewriteinfo` data, which shows which files are frozen or managed by CMDaemon:

```bash
cmsh -c "device; filewriteinfo" > bcm/filewriteinfo.txt
```

---

## 2. NVIDIA Mission Control (NMC)

### 2.1 Load each cluster context

BCM normally uses `k8s-system-admin` and `k8s-system-user` as the cluster names.

```bash
module avail 2>&1 | grep -i kubernetes > nmc/k8s-modules.txt
```

### 2.2 Releases and workloads

Run commands in 2.2, 3.1, 3.2, 3.3 in each kubernetes cluster (k8s-system-admin, k8s-system-user, etc.) 

```bash
module load kubernetes/k8s-system-admin           # repeat everything in 2.2 for k8s-system-user modeule for the second pass
CL=k8s-system-admin   # set to "k8s-system-user" for the second pass
helm list -A -o yaml > nmc/$CL-helm-releases.yaml
for r in $(helm list -A -o json | jq -r '.[]|"\(.namespace)/\(.name)"'); do
  ns=${r%/*}; n=${r#*/}
  helm get values $n -n $ns -a > nmc/$CL-values-$ns-$n.yaml
  helm get manifest $n -n $ns > nmc/$CL-manifest-$ns-$n.yaml
done
kubectl get pods -A -o wide > nmc/$CL-pods.txt
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}' > nmc/$CL-images.txt
kubectl get pvc,pv -A -o wide > nmc/$CL-storage.txt
```

---

## 3. Kubernetes (each cluster, including the one running the in-house scheduler)

### 3.1 Version and nodes

```bash
kubectl version -o yaml > k8s/$CL-version.yaml
kubectl get nodes -o wide --show-labels > k8s/$CL-nodes.txt
kubectl describe nodes > k8s/$CL-nodes-describe.txt
kubectl get nodes -o json | jq '.items[]|{n:.metadata.name,taints:.spec.taints,alloc:.status.allocatable}' > k8s/$CL-taints-allocatable.json
cmsh -c "kubernetes; use $CL; show" > k8s/bcm-k8s-$CL.txt
```

### 3.2 Full object dump

```bash
kubectl get all,cm,secret,sa,role,rolebinding,ingress,netpol,pdb,hpa,resourcequota,limitrange -A -o yaml > k8s/$CL-namespaced-objects.yaml
kubectl get ns,clusterrole,clusterrolebinding,sc,pv,priorityclass,runtimeclass,ingressclass,apiservice -o yaml > k8s/$CL-cluster-objects.yaml
kubectl get crd -o yaml > k8s/$CL-crds.yaml
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration -o yaml > k8s/$CL-webhooks.yaml
kubectl get events -A --sort-by=.lastTimestamp > k8s/$CL-events.txt
```

### 3.3 GPU and Network Operator, DRA

```bash
helm list -A | grep -Ei 'gpu-operator|network-operator|dra' > k8s/$CL-operators.txt
kubectl get clusterpolicy -o yaml > k8s/$CL-gpu-clusterpolicy.yaml
kubectl get nicclusterpolicy -o yaml > k8s/$CL-nic-clusterpolicy.yaml 2>&1
kubectl get pods -n nvidia-dra-driver-gpu -o yaml > k8s/$CL-dra-pods.yaml 2>&1
kubectl get resourceclaims,resourceclaimtemplates,deviceclasses -A -o yaml > k8s/$CL-dra-objects.yaml 2>&1
```

---

## 4. In-house scheduler (highest risk item)

A custom scheduler is the component most likely to break on a Kubernetes version change. It can break through the scheduler framework API version, removed APIs, RBAC, or leader-election leases.

### 4.1 Collection

```bash
NS=<scheduler-namespace>; SCHED=<scheduler-name>
kubectl -n $NS get deploy,sts,ds,pod,svc,cm,secret,sa -o yaml > scheduler/objects.yaml
kubectl -n $NS get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}' > scheduler/images.txt
kubectl -n $NS logs deploy/<scheduler-deploy> --all-containers --tail=5000 > scheduler/logs.txt
kubectl -n $NS get cm -o yaml | grep -A50 KubeSchedulerConfiguration > scheduler/sched-config.txt   # note the apiVersion
kubectl get lease -A | grep -i sched > scheduler/leases.txt
kubectl get clusterrole,clusterrolebinding -o yaml | grep -B5 -A30 $SCHED > scheduler/rbac.txt
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.spec.schedulerName}{"\n"}{end}' | grep -v default-scheduler > scheduler/pods-using-custom-sched.txt
kubectl get pods -A --field-selector=status.phase=Pending -o wide > scheduler/pending-pods.txt
kubectl get crd | grep -i <scheduler-api-group> > scheduler/crds.txt
kubectl get priorityclass -o yaml > scheduler/priorityclasses.yaml
```

---

## 5. Operating system (head nodes and each node category)

### 5.1 BCM Head node

```bash
cat /etc/os-release; uname -a > os/head-release.txt
dpkg -l > os/head-packages.txt 2>/dev/null || rpm -qa > os/head-packages.txt
apt-mark showhold > os/head-held-pkgs.txt 2>/dev/null
cat /proc/cmdline > os/head-cmdline.txt
df -hT; lsblk; cat /etc/fstab > os/head-storage.txt
mount > os/head-mounts.txt
ip -br a; ip r; cat /etc/netplan/*.yaml > os/head-network.txt 2>&1
systemctl --failed; systemctl list-unit-files --state=enabled > os/head-services.txt
sysctl -a > os/head-sysctl.txt 2>/dev/null
chronyc tracking > os/head-time.txt 2>&1
crontab -l; ls /etc/cron.d > os/head-cron.txt 2>&1
tar czf os/head-etc.tgz /etc
```

### 5.2 Each compute and Kubernetes node category

Run this once per category:

```bash
cmsh -c "device; pexec -t PhysicalNode -j -- cat /etc/os-release | grep PRETTY" > os/PhysicalNode-release.txt
cmsh -c "device; pexec -t PhysicalNode -j -- uname -r" > os/PhysicalNode-release-uname.txt
cmsh -c "device; pexec -t PhysicalNode -j -- df -h /" > os/PhysicalNode-health1.txt
cmsh -c "device; pexec -t PhysicalNode -j -- uptime" > os/PhysicalNode-health2.txt
cmsh -c "device; pexec -t PhysicalNode -j -- systemctl --failed --no-legend" > os/PhysicalNode-health3.txt
cmsh -c "device; pexec -t PhysicalNode -j -- cat /proc/cmdline" > os/PhysicalNode-cmdline.txt
```

---

## 6. Drivers and firmware

### 6.1 GPU, CUDA and DCGM

```bash
cmsh -c "device; pexec -t PhysicalNode -j -- nvidia-smi --query-gpu=index,name,driver_version,vbios_version,serial --format=csv,noheader" > drivers/PhysicalNode-gpu.txt
cmsh -c "device; pexec -t PhysicalNode -j -- nvidia-smi -q | grep -Ei 'driver|cuda|fabric|state'" > drivers/PhysicalNode-gpu-q.txt
cmsh -c "device; pexec -t PhysicalNode -j -- dcgmi diag -r 1" > drivers/PhysicalNode-dcgmi.txt
cmsh -c "device; pexec -t PhysicalNode -j -- dpkg -l | grep -Ei 'nvidia|cuda|dcgm|fabricmanager|imex|doca|mlnx'" > drivers/PhysicalNode-pkgs.txt
cmsh -c "device; pexec -t PhysicalNode -j -- systemctl is-active nvidia-fabricmanager nvidia-imex nvidia-persistenced" > drivers/PhysicalNode-services.txt
```

### 6.2 NIC, DOCA/OFED and InfiniBand

```bash
cmsh -c "device; pexec -t PhysicalNode -j -- ofed_info -s" > drivers/PhysicalNode-nic-ofed.txt
cmsh -c "device; pexec -t PhysicalNode -j -- mlxfwmanager --query" > drivers/PhysicalNode-nic-mlx.txt
cmsh -c "device; pexec -t PhysicalNode -j -- ibstat | grep -E 'CA |State|Rate|Firmware'" > fabric/PhysicalNode-ibstat.txt
```

### 6.3 Firmware baseline and diagnostics

```bash
cmsh -c "device; pexec -t PhysicalNode -j -- ipmitool mc info" > drivers/PhysicalNode-bmc1.txt
cmsh -c "device; pexec -t PhysicalNode -j -- ipmitool sel elist | tail -50" > drivers/PhysicalNode-bmc2.txt
cmsh -c "device; firmware info" > fw/firmware-info.txt
cmsh -c "device; firmware status -t PhysicalNode" > fw/firmware-detailed.txt
```

## 7. Package and validate the bundle

```bash
cmsh -c "device; foreach -v * (latesthealthdata)" > bcm/health-final.txt
tar czf /cm/shared/backups/$(basename $OUT).tgz -C $(dirname $OUT) $(basename $OUT)
sha256sum /cm/shared/backups/$(basename $OUT).tgz
```

Copy the bundle **off-cluster** and share with your NVIDIA CBU SA via email or using the link to the file share provided earlier.
