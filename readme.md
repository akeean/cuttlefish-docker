This guide documents the final, working architecture for running Cuttlefish (Android) on a Linux host (Littlefoot) using a host-integrated Docker model. It captures the specific technical hurdles bypassed to achieve a working WebRTC display and successful APK installation.

🏛️ 1. Architecture & System Integration

Unlike standard "contained" Docker images, Cuttlefish requires deep integration with the host kernel to handle nested virtualization.

    Kernel Requirements: The host must have kvm enabled and specific vsock modules loaded.

        Command: sudo modprobe vsock vhost_vsock vhost_net vhci_hcd

    Networking Strategy: We utilize --net host. This is the single most important choice. It allows the container to share the host's IP stack, bypassing the flaky Docker bridge which often breaks WebRTC signaling and creates "offline" ADB devices.

    The Operator Conflict: On many systems, a local "operator" service may squat on ports. The script must force-kill this (sudo pkill -9 operator) to ensure the signaling server can bind to port 8443.

📂 2. Artifact Sourcing (The "Build ID" Match)

Cuttlefish will fail to boot if the host tools and the system image aren't perfectly synced.

    Source: ci.android.com.

    Target: aosp_cf_x86_64_only_phone-userdebug.

    The Pair: You must download both the Image Zip and the cvd-host_package.tar.gz from the same Build ID.

    Extraction: Unzip the image into the same directory where you extracted the host package.

🖥️ 3. Graphics: The SwiftShader Pivot

Passing a physical GPU into a nested VM inside a Docker container is a primary point of failure for WebRTC display.

    The Fix: Use --gpu_mode=guest_swiftshader.

    Why: This forces CPU-based software rendering for the Android guest. It is slightly slower but guarantees that the WebRTC stream will actually initialize and display the UI regardless of host driver issues.

🌐 4. WebRTC & Browser Interaction

The display is accessed via a browser-based signaling server.

    URL: https://localhost:8443 (The port must be explicitly set via --webrtc_sig_server_port=8443).

    The "Double Security" Hazard:

        The browser will flag the site as unsafe (self-signed cert). You must click Advanced -> Proceed.

        The signaling server may require a second "Accept" within the UI to start the stream.

    Input Latency: If the WebRTC mouse/keyboard feels unresponsive, use the adb shell input command via the host terminal instead.

📱 5. ADB Connectivity & APK Deployment

Because of the host networking and virtualized nature, ADB often behaves erratically.

    The Serial Trap: ADB may see 127.0.0.1:6520 and localhost:6520 simultaneously.

        The Fix: Always connect and target explicitly: adb connect localhost:6520 followed by adb -s localhost:6520 <command>.

    Installing APKs:

        Ensure the device is device status in adb devices.

        Run: adb -s localhost:6520 install your_app.apk.

        If it hangs, check cvd-docker logs to ensure the adbd service inside the guest hasn't crashed.

🛠️ 6. Troubleshooting & Session Hygiene

Cuttlefish creates "sockets" and "runtime files" on the host disk that do not disappear when the container stops. This prevents subsequent boots.

    The "Clean" Protocol: Before every fresh start, you must wipe the runtime directories:

        rm -rf ~/cuttlefish_images/cuttlefish_runtime

        rm -rf ~/cuttlefish_images/cf_avd_0

        Socket Cleanup: find ~/cuttlefish_images -type s -delete

    Permission Denied: If launch_cvd fails to access /dev/kvm inside the container, the script must run a root chmod 666 /dev/kvm /dev/vsock immediately after docker run.

🚀 Summary of the "Working Path"

    Host: Load kernel modules + Kill conflicting operators.

    Docker: Launch with --privileged and --net host.

    Guest: Boot with guest_swiftshader and daemon mode.

    Display: Access via HTTPS on port 8443.

    Tunnel: (For Expo) adb reverse tcp:8081 tcp:8081.

Would you like me to help you draft the README.md for that GitHub repository so others can reproduce this Littlefoot setup?
