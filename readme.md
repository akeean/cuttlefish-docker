# Cuttlefish in Docker: Complete Orchestration & Setup Guide

This project provides a complete framework for building and running Cuttlefish (Android Open Source Project) within a Docker container. It uses a Host-Integrated architecture optimized for Linux environments (like SFF PCs), leveraging the host's network stack and kernel modules to ensure stable WebRTC streaming and reliable nested virtualization.

# 🛠️ Phase 1: Initial Installation & Build

Before using the cvd-docker script, you must prepare the host and build the base container image.

## 1. Install Dependencies

Your host machine needs the standard Cuttlefish host tools and Docker:
Bash

    sudo apt install -y git devscripts config-package-dev debhelper-compat golang curl
    git clone https://github.com/google/android-cuttlefish.git
    cd android-cuttlefish
    debuild -i -us -uc -b
    sudo dpkg -i ../*.deb
    sudo apt install -f
    sudo adduser $USER cvdnetwork
    sudo adduser $USER kvm

### Reboot your machine after adding users

## 2. Build the Docker Image

From the root of this repository, build the orchestration image:
Bash

    docker build -t cuttlefish-orchestration .

## 3. Install the Orchestration Script

Move the cvd-docker script to your system path so you can call it from anywhere:
Bash

    sudo mv cvd-docker /usr/local/bin/cvd-docker
    sudo chmod +x /usr/local/bin/cvd-docker

# 🏛️ Phase 2: System Integration

## 1. Kernel Module Requirements

Cuttlefish requires direct kernel interaction. The script handles this, but your host must support:

    vsock / vhost_vsock: For high-speed Guest-to-Host communication.

    vhost_net: For optimized networking offloading.

    vhci_hcd: For virtual USB redirection.

## 2. Networking Strategy: --net host

This project bypasses Docker's bridge networking. By using the host network stack, the container shares the host's IP. This is the only reliable way to ensure WebRTC signaling and media streams remain synchronized without NAT-related failures.

# 📂 Phase 3: Sourcing System Images (The Build ID Rule)

Cuttlefish will fail to boot if the host tools and the system image aren't perfectly synced.

Navigate to:

    ci.android.com.

Target: aosp_cf_x86_64_only_phone-userdebug.

The Pair: Download the Image Zip and the cvd-host_package.tar.gz from the same Build ID (pick a green build that had no errors).

Extraction: Extract both into ~/cuttlefish_images. This directory is mounted into the container as a persistent volume.

    ~/cuttlefish_images/
    ├── bin/                   <-- From Host Package (contains launch_cvd, etc.)
    ├── etc/                   <-- From Host Package
    ├── lib64/                 <-- From Host Package
    ├── metadata/              <-- From Host Package
    ├── usr/                   <-- From Host Package
    ├── boot.img               <-- From System Image Zip
    ├── initrd.img             <-- From System Image Zip
    ├── system.img             <-- From System Image Zip
    ├── userdata.img           <-- From System Image Zip
    ├── vendor.img             <-- From System Image Zip
    ├── vbmeta.img             <-- From System Image Zip
    └── ... (other .img and .json files from the Zip)

# 📜 Phase 4: Script Usage & Commands

Use the cvd-docker command to manage your daily sessions.

    cvd-docker start

Injects kernel modules and kills conflicting operator services.

Launches a privileged container with SwiftShader (CPU-based rendering) to ensure the WebRTC display works regardless of your physical GPU drivers.

    cvd-docker stop

Executes a graceful stop_cvd inside the guest. Always use this instead of docker stop to avoid corrupting the virtual Android disk.

    cvd-docker clean

The "Nuke" Option. If a launch fails, run this to wipe stale Unix sockets (.sock), lock files, and runtime directories (cuttlefish_runtime) that prevent new sessions from starting.

    cvd-docker logs

Streams the launcher log. Look for "VNC/WebRTC server started" to confirm a successful boot.

# 🖥️ Phase 5: WebRTC & Development Tunneling

## 1. Accessing the UI

    URL: https://localhost:8443

You must manually accept the self-signed certificate in your browser (Advanced -> Proceed).

## 2. ADB Connectivity

Because of the host networking model, ADB may see duplicate serials. Always target explicitly: adb -s localhost:6520 [command]

## 3. The Expo/Metro Tunnel

To make an app inside Cuttlefish talk to a development server on your host: adb -s localhost:6520 reverse tcp:8081 tcp:8081

# 🛡️ Troubleshooting & Safety

Stale States: If you get "Cannot lock instance," run cvd-docker clean.

System Hangs: If the host behaves erratically after a crash, reboot the physical PC to reset the vsock kernel state.

Security: This setup uses --privileged and --net host. It is for Local Development Only. 

#### Do not expose ports 8443 or 6520 to the public internet.
