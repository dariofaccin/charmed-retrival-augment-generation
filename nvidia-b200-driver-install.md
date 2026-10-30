# NVIDIA driver installation – HGX B200 host (`10.241.201.15`)

- **Host:** `buzzsaw`, Ubuntu 24.04.5 LTS, kernel `6.8.0-147-generic`
- **GPUs:** 8× NVIDIA B200 (PCI device `10de:2901`), NVSwitch fabric managed through ConnectX-7
- **Driver:** `580.178.04`, Ubuntu `580-server-open` branch. Blackwell requires the open kernel modules.

All commands were run as `ubuntu` on the host. Use `sudo` where shown.

## 1. Pre-checks

```bash
lsb_release -ds; uname -r
lspci | grep -i nvidia                 # 8x "3D controller: NVIDIA Corporation Device 2901"
mokutil --sb-state                     # SecureBoot disabled
lsmod | grep -E "nouveau|nvidia"       # nouveau was loaded, no nvidia module
lspci | grep -i mellanox               # ConnectX-7 (needed by NVLSM / Fabric Manager)
```

## 2. Install driver, Fabric Manager and NSCQ (Ubuntu archive)

Use Ubuntu's prebuilt, signed kernel modules (no DKMS). The `-generic` meta package tracks kernel updates.

```bash
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  linux-modules-nvidia-580-server-open-generic \
  nvidia-headless-no-dkms-580-server-open \
  nvidia-utils-580-server \
  nvidia-fabricmanager-580 \
  libnvidia-nscq-580
```

The packages blacklist `nouveau` through `/lib/modprobe.d/nvidia-graphics-drivers.conf`.

## 3. Install NVLSM (NVLink Subnet Manager) from the NVIDIA CUDA repo

HGX B200 Fabric Manager requires `nvlsm`, which is not in the Ubuntu archive. Add the CUDA repo and pin it so that **only** `nvlsm` comes from it. Everything else, including the driver, stays on Ubuntu packages.

```bash
cd /tmp
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb && rm cuda-keyring_1.1-1_all.deb

sudo tee /etc/apt/preferences.d/nvidia-cuda-repo-pin >/dev/null <<'EOF'
Package: *
Pin: origin developer.download.nvidia.com
Pin-Priority: -1

Package: nvlsm libibumad3* infiniband-diags*
Pin: origin developer.download.nvidia.com
Pin-Priority: 500
EOF

sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y nvlsm
```

## 4. Install `ibstat` (required by Fabric Manager start script)

The first Fabric Manager start failed with `"ibstat" command not found!`. The fix:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y infiniband-diags
```

> Tip: on a fresh host, install `infiniband-diags` together with `nvlsm` in step 3. Fabric Manager then starts cleanly on first boot.

## 5. Rebuild initramfs and reboot (swap nouveau for nvidia)

```bash
sudo update-initramfs -u -k all
sudo systemctl reboot
```

## 6. Load RDMA / InfiniBand kernel modules (now and at boot)

```bash
printf "ib_umad\nrdma_cm\nrdma_ucm\nib_ipoib\n" | sudo tee /etc/modules-load.d/rdma.conf
for m in ib_umad rdma_cm rdma_ucm ib_ipoib; do sudo modprobe $m; done
sudo systemctl restart systemd-modules-load

lsmod | grep -E "^(ib_umad|rdma_cm|rdma_ucm|ib_ipoib|nvidia|nouveau) "
```

## 7. Start Fabric Manager

The Fabric Manager service (`nvidia-fabricmanager`) is enabled by the package. After installing `ibstat`, restart it:

```bash
sudo systemctl restart nvidia-fabricmanager
systemctl is-active nvidia-fabricmanager nvidia-persistenced
sudo journalctl -u nvidia-fabricmanager -b --no-pager | tail
# expect: "Successfully configured all the available GPUs and NVSwitches to route NVLink traffic."
```

## 8. Verify GPU availability

```bash
# Driver and GPUs
nvidia-smi
nvidia-smi --query-gpu=index,name,driver_version,memory.total --format=csv

# NVLink fabric registration: all 8 GPUs should be "Completed / Success"
nvidia-smi -q | grep -A2 "^    Fabric" | grep -E "State|Status" | sort | uniq -c

# NVLink topology: expect NV18 between every GPU pair
nvidia-smi topo -m

# Per-link status (18 links x 50 GB/s per GPU)
nvidia-smi nvlink -s -i 0
```

Results: 8× NVIDIA B200, driver `580.178.04`, 183359 MiB each. Fabric state was `Completed`/`Success` on all GPUs, with `NV18` full-mesh topology.

## 9. Kubernetes check (GPU Operator)

```bash
kubectl get pods -n gpu-operator-resources
kubectl get node buzzsaw -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'   # 8
kubectl get clusterpolicy -o jsonpath='{.items[0].status.state}{"\n"}'              # ready
```

Right after cluster start, `FailedCreatePodSandBox: no runtime for "nvidia" is configured` events are expected. They clear once `nvidia-container-toolkit-daemonset` configures containerd.
