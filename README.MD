# Unitree Go2 EDU (Jetson Orin NX) — RL Policy Deployment Environment Setup

This guide documents how to access the onboard **NVIDIA Jetson Orin NX** inside the Unitree Go2 EDU expansion dock, establish internet connectivity, and build the C++ reinforcement learning (RL) deployment stack (`unitree_sdk2`, `ONNX Runtime GPU`, and `unitree_cpp_deploy`) for policies trained in **Isaac Lab** and exported to ONNX.

> **CRITICAL WARNING:** **DO NOT** reflash the Jetson Orin, run `sudo apt dist-upgrade`, or upgrade JetPack / L4T packages. Doing so will overwrite Unitree's custom carrier-board device trees, CycloneDDS configurations, and preinstalled system services.

---

## 1. Network Architecture & SSH Setup

The Go2 EDU contains an internal Gigabit Ethernet switch on the `192.168.123.0/24` subnet connecting all onboard compute modules.

### Internal Subnet Map (`192.168.123.0/24`)

| Device | IP Address | Default Credentials | Role |
| :--- | :--- | :--- | :--- |
| **Go2 Motion MCU** | `192.168.123.161` | N/A | Low-level motor/IMU controller (500 Hz DDS) |
| **Jetson Orin NX (Default)** | `192.168.123.18` | `unitree` / `123` | Expansion AI compute (check `.15` if `.18` fails) |
| **Developer Laptop** | `192.168.123.100` | Local User | Static IP assigned to laptop Ethernet adapter |

### Step 1.1: Configure Laptop Static IP & SSH
1. Power on the Go2 EDU (short press + 2-second hold) and wait ~2 minutes for the solid green head LED.
2. Connect an Ethernet cable from your laptop to the RJ45 port on the Go2 expansion dock.
3. Configure your laptop's wired interface with a static IP in `192.168.123.x/24`:
   * **Windows (Administrator PowerShell):**
     ```powershell
     New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.123.100 -PrefixLength 24
     Set-NetIPInterface -InterfaceIndex 13 -Dhcp Disabled
     ```
   * **Linux (Ubuntu):**
     ```bash
     sudo nmcli con add type ethernet ifname eth0 con-name go2-lan ip4 192.168.123.100/24
     sudo nmcli con up go2-lan
     ```
4. Copy your SSH public key for passwordless login:
   ```bash
   ssh-copy-id unitree@192.168.123.18


## 2. Providing Internet Access to the Jetson Orin 

To allow the Orin to clone repositories and download dependencies without a physical Wi-Fi dongle, share your laptop's internet connection over the Ethernet cable:

On Windows Laptop: 
1. Open Network Connections (ncpa.cpl), right-click your Wi-Fi adapter $\rightarrow$ Properties $\rightarrow$ Sharing tab $\rightarrow$ check Allow other network users to connect through this computer's Internet connection and select Ethernet.
2. On Windows Administrator PowerShell: 
Enable direct routing/NAT for the subnet:
```powershell
Set-NetIPInterface -InterfaceIndex 13 -Forwarding Enabled
Get-NetNat | Remove-NetNat -Confirm:$false
New-NetNat -Name "Go2OrinNAT" -InternalIPInterfaceAddressPrefix "192.168.123.0/24"
```

3. On the Jetson Orin (ssh unitree@192.168.123.18): Point the default gateway to your laptop (192.168.123.100) and set DNS:
```Bash
sudo ip route replace default via 192.168.123.100 dev eth0
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
ping -c 3 github.com
```


### 3. Environment & Dependency Versions

| Component | Version / Specification | Notes |
|---|---|---|
| Hardware | NVIDIA Jetson Orin NX (16GB) | aarch64 architecture |
| OS / JetPack | Ubuntu 20.04 LTS (JetPack 5.1.1 / L4T R35.3.1) | Preinstalled factory image |
| CUDA / cuDNN | CUDA 11.4 / cuDNN 8.6+ | Located at `/usr/local/cuda-11.4` |
| unitree_sdk2 | main branch (v2.0.0) | Installed system-wide to `/usr/local` |
| ONNX Runtime | v1.16.0 (linux-aarch64-gpu) | Built against CUDA 11.4 + |



## 4. Install and Build Steps

### Step 4.1: Install System Development Libraries

SSH into the Orin and run:

```bash
sudo apt update && sudo apt install -y \
    build-essential \
    cmake \
    git \
    git-lfs \
    libyaml-cpp-dev \
    libboost-all-dev \
    libeigen3-dev \
    libspdlog-dev \
    libfmt-dev

git lfs install
```

### Step 4.2: Build and Install `unitree_sdk2`

```bash
cd ~
git clone https://github.com/unitreerobotics/unitree_sdk2.git
cd unitree_sdk2

mkdir -p build && cd build
cmake .. -DBUILD_EXAMPLES=OFF
sudo make install -j$(nproc)
```

### Step 4.3: Download ONNX Runtime GPU (`v1.16.0`) & Build Deploy Controller

```bash
cd ~

wget https://github.com/csukuangfj/onnxruntime-libs/releases/download/v1.16.0/onnxruntime-linux-aarch64-gpu-1.16.0.tar.bz2

git clone https://github.com/wty-yy/unitree_cpp_deploy.git
cd unitree_cpp_deploy

mkdir -p deploy/thirdparty

tar -xjvf ~/onnxruntime-linux-aarch64-gpu-1.16.0.tar.bz2 \
    -C deploy/thirdparty/

cd deploy/robots/go2

cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)
```

### Step 4.4: Verify Build

Test the compiled binary locally without touching motors:

```bash
./build/go2_ctrl --help
```

## 5. Known Issues & Fixes

1. **`ros:foxy(1) noetic(2) ?` SSH Hang:** Unitree's default `~/.bashrc` includes an interactive `read` prompt that hangs non-interactive SSH sessions and VS Code Remote-SSH. Fix by replacing the `fishros initialize` block with a static `source /opt/ros/foxy/setup.bash`.

2. **Missing Git LFS Files:** Cloning repositories containing `.onnx` models without `git-lfs` downloads pointer text files instead of binary weights. Always run:

   ```bash
   sudo apt install git-lfs
   git lfs install
   git lfs pull
   ```

3. **CycloneDDS Interface Binding:** When running the controller on the real robot, always specify the robot's internal Ethernet interface explicitly so multicast traffic routes correctly:

   ```bash
   ./build/go2_ctrl -n eth0
   ```