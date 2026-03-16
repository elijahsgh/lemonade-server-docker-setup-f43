# lemonade-server-docker-setup-f43

## Purpose

This is my repository for tracking what I've done to get [Lemonade Server](https://lemonade-server.ai/) working on my Fedora 43 system.

## Requirements

### Increase limits

Add to /etc/security/limits.conf

```
* soft memlock unlimited
* hard memlock unlimited
```

### amd npu driver

I believe at this time the most recent Fedora 43 is kernel 6.19 and comes with the appropriate drivers.

If the drivers are correct you should see something like this:

```
❯ sudo dmesg | grep xdna
[    0.906480] amdxdna 0000:be:00.1: [drm] Load firmware amdnpu/17f0_11/npu.sbin
[    0.906498] amdxdna 0000:be:00.1: enabling device (0000 -> 0002)
[    1.065386] [drm] Initialized amdxdna_accel_driver 0.6.0 for 0000:be:00.1 on minor 0
```

### xrt

I am using the xrt from the xanderlent copr available here [xanderlent amd-npu-driver](https://copr.fedorainfracloud.org/coprs/xanderlent/amd-npu-driver/)

You may need to `sudo ln -s /usr/xrt/lib64/libxrt_core.so.2 /usr/xrt/lib/libxrt_core.so.2` if you see an error about that library not being found (lib vs lib64).

Verifying xrt (if using this copr, otherwise the paths are different):

```
❯ source /usr/xrt/setup.sh 
Autocomplete enabled for the xrt-smi command
Autocomplete enabled for the xbmgmt command
XILINX_XRT        : /usr/xrt
PATH              : /usr/xrt/bin:/usr/share/Modules/bin:/usr/local/bin:/usr/bin
LD_LIBRARY_PATH   : /usr/xrt/lib
PYTHONPATH        : /usr/xrt/python

🌐 ryzenaimax in ~ 
❯ sudo -E /usr/xrt/bin/xrt-smi examine
System Configuration
  OS Name              : Linux
  Release              : 7.0.0-0.rc3.227.vanilla.fc43.x86_64
  Machine              : x86_64
  CPU Cores            : 32
  Memory               : 127931 MB
  Distribution         : Fedora Linux 43 (Workstation Edition)
  GLIBC                : 2.42
  Model                : MS-S1 MAX
  BIOS Vendor          : American Megatrends International, LLC.
  BIOS Version         : 1.06

XRT
  Version              : 2.19.0
  Branch               : 
  Hash                 : 
  Hash Date            : 2025-04-25 00:00:00
  virtio-pci           : unknown, unknown
  amdxdna              : unknown, unknown
  NPU Firmware Version : 1.1.2.65

Device(s) Present
|BDF             |Name          |
|----------------|--------------|
|[0000:be:00.1]  |RyzenAI-npu5  |
```

*Note that I'm using kernel 7.0 here, but you should be able to use 6.19*

## Docker

I use the following docker file with podman (of course) based on the original in the [Lemonade Server repo](https://github.com/lemonade-sdk/lemonade/blob/main/Dockerfile).

The only real significant change is the inclusion of libatomic1.

```
# ==============================================================
# # 1. Build stage — compile lemonade C++ binaries
# # ============================================================
FROM ubuntu:24.04 AS builder

# Avoid interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    ninja-build \
    libssl-dev \
    pkg-config \
    libdrm-dev \
    git \
    && rm -rf /var/lib/apt/lists/*

# Copy source code
COPY . /app
WORKDIR /app

# Build the project
RUN rm -rf build && \
    cmake --preset default && \
    cmake --build --preset default

# Debug: Check build outputs
RUN echo "=== Build directory contents ===" && \
    ls -la build/ && \
    echo "=== Checking for resources ===" && \
    find build/ -name "*.json" -o -name "resources" -type d

# # ============================================================
# # 2. Runtime stage — small, clean image
# # ============================================================
FROM ubuntu:24.04

# Install runtime dependencies only
RUN apt-get update && apt-get install -y \
    libcurl4 \
    curl \
    libssl3 \
    zlib1g \
    libdrm2 \
    vulkan-tools \
    libvulkan1 \
    unzip \
    libgomp1 \
    libatomic1 \
    software-properties-common \
    && rm -rf /var/lib/apt/lists/*

RUN add-apt-repository -y ppa:amd-team/xrt

# Download FLM and install it
RUN curl -L -o fastflowlm.deb https://github.com/FastFlowLM/FastFlowLM/releases/download/v0.9.35/fastflowlm_0.9.35_ubuntu24.04_amd64.deb && \
    apt install -y ./fastflowlm.deb && \
    rm fastflowlm.deb

# Create application directory
WORKDIR /opt/lemonade

# Copy built executables and resources from builder
COPY --from=builder /app/build/lemonade-router ./lemonade-router
COPY --from=builder /app/build/lemonade-server ./lemonade-server
COPY --from=builder /app/build/resources ./resources

# Make executables executable
RUN chmod +x ./lemonade-router ./lemonade-server

# Create necessary directories
RUN mkdir -p /opt/lemonade/llama/cpu \
    /opt/lemonade/llama/vulkan \
    /root/.cache/huggingface

# Expose default port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/live || exit 1

# Default command: start server in headless mode
CMD ["./lemonade-server", "serve", "--no-tray", "--host", "0.0.0.0"]
```

## Quadlet

I run as a user, but you can install this in system instead.

As a user, in `./.config/containers/systemd/lemonade-server.container`

```
[Unit]
Description=Lemonade Server
After=network-online.target

[Container]
ContainerName=lemonade-server
Image=localhost/lemonade-server-with-fastflowlm:latest
Exec=./lemonade-server serve --host 0.0.0.0 --port 8000 --max-loaded-models 5 --ctx-size 32768

AddDevice=/dev/kfd
AddDevice=/dev/dri
AddDevice=/dev/accel/accel0

GroupAdd=video
AddCapability=CAP_SYS_PTRACE
SecurityLabelDisable=true
SeccompProfile=unconfined
Ulimit=memlock=-1:-1

Volume=<SET TO YOUR ABSOLUTE PATH>/lemonade-docker/huggingface-cache:/root/.cache/huggingface:Z
Volume=<SET TO YOUR ABSOLUTE PATH>/lemonade-docker/lemonade-cache:/root/.cache/lemonade:Z
Volume=<SET TO YOUR ABSOLUTE PATH>/lemonade-docker/lemonade-llama:/opt/lemonade/llama:Z
Volume=<SET TO YOUR ABSOLUTE PATH>/lemonade-docker/flm-data:/root/.config/flm:Z

PublishPort=8000:8000

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

```
systemctl --user daemon-reload
systemctl --user start lemonade-server
journalctl --user logs lemonade-server
```

### linger user

If using the quadlet as a user, you need to enable linger for that user.

```
loginctl enable-linger myuser
loginctl show-user myuser --property=Linger
```