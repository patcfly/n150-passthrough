# Intel N150 GPU Hardware PCIE Passthrough and Acceleration for inference in Frigate docker on Debian 13 VM on Proxmox-VE 9 using IOMMU and SRIOV

This guide documents the steps taken to successfully configure Intel N150 Integrated GPU for hardware acceleration in Frigate NVR running in Docker on Debian VM on Proxmox VE 9.

## Table of Contents
- [Intel N150 GPU Hardware PCIE Passthrough and Acceleration for inference in Frigate docker on Debian 13 VM on Proxmox-VE 9 using IOMMU and SRIOV](#intel-n150-gpu-hardware-pcie-passthrough-and-acceleration-for-inference-in-frigate-docker-on-debian-13-vm-on-proxmox-ve-9-using-iommu-and-sriov)
  - [Table of Contents](#table-of-contents)
  - [Goal](#goal)
  - [Use Case](#use-case)
  - [References](#references)
  - [System Configuration](#system-configuration)
  - [Prerequisites](#prerequisites)
    - [1. BIOS Configuration (Required for GPU Passthrough)](#1-bios-configuration-required-for-gpu-passthrough)
    - [2. VM Configuration in Proxmox:](#2-vm-configuration-in-proxmox)
    - [3. Verify GPU access in Debian 13 VM](#3-verify-gpu-access-in-debian-13-vm)
    - [2. VM OS Kernel Upgrade (Critical for Intel GPU Support)](#2-vm-os-kernel-upgrade-critical-for-intel-gpu-support)
    - [3. Intel GPU Drivers and Firmware](#3-intel-gpu-drivers-and-firmware)
    - [4. Docker](#4-docker)
    - [5. User Permissions](#5-user-permissions)
    - [6. Verify GPU Access](#6-verify-gpu-access)
  - [Docker Configuration](#docker-configuration)
    - [Docker Compose Setup](#docker-compose-setup)
  - [Frigate Configuration](#frigate-configuration)
    - [Key Configuration Elements](#key-configuration-elements)
      - [Detector Configuration (OpenVINO with Intel GPU)](#detector-configuration-openvino-with-intel-gpu)
      - [Model Configuration (YOLOv9)](#model-configuration-yolov9)
      - [Camera Configuration](#camera-configuration)
  - [Troubleshooting Steps Taken](#troubleshooting-steps-taken)
    - [1. Initial Container Crashes](#1-initial-container-crashes)
    - [2. Model File Access Issues](#2-model-file-access-issues)
    - [3. Configuration Validation Errors](#3-configuration-validation-errors)
    - [4. GPU Telemetry Issues (Partial Success)](#4-gpu-telemetry-issues-partial-success)
  - [Final Working Configuration](#final-working-configuration)
    - [Hardware Acceleration Status](#hardware-acceleration-status)
    - [Performance Results](#performance-results)
  - [Key Learnings](#key-learnings)
  - [Files Structure](#files-structure)
  - [Verification Commands](#verification-commands)
  - [Notes](#notes)

## Goal

My goal was to use the integrated GPU for hardware acceleration of Frigate video inference models without the need for buying a purpose specific hardware AI accelerator such as a Coral or Hailo pcie card. I did not want to use a consumer-grade solution due to privacy concerns and I wanted to use this as an opportunity to learn more about hardware passthrough and AI models.

## Use Case
My use case: I plan on running a single PoE security camera at a location with very high electricity costs, so keeping power consumption down to a minimum is ideal. I wanted to verify hardware acceleration would work before buying the necesary equipment (PoE camera,and PoE switch or injector), so I used a spare USB webcam I already owned.

## References

I read the following posts, blogs, and official frigate documentation, as well as extensive use of Github Copilot to get this working:

 - [Frigate With an Intel iGPU & YOLO-NAS](https://davidjusto.com/articles/frigate-openvino-igpu/)
 - [Not able to passthrough iGPU to docker apps (N150) - TRUENAS Support](https://forums.truenas.com/t/not-able-to-passthrough-igpu-to-docker-apps-n150/29856)
 - [12th Gen Intel UHD 770 (Alder Lake) iGPU support - TRUENAS Forums](https://www.truenas.com/community/threads/12th-gen-intel-uhd-770-alder-lake-igpu-support.111739/#post-778043)
 - [Intel N150 GPU passthrough - Proxmox forums](https://forum.proxmox.com/threads/intel-n150-gpu-passthrough.160477/)
 - [Intel N150 Hardware Transcoding on Ubuntu LTS 24.04.1 LTS - Plex Forums](https://forums.plex.tv/t/intel-n150-hardware-transcoding-on-ubuntu-lts-24-04-1-lts/899721)

## System Configuration
- **Hypervisor**: pve-manager/9.0.10/deb1ca707ec72a89
- **Hypervisor Kernel**: 6.14.11-2-pve
- **VM OS**: Debian 13 VM running on Proxmox VE 9
- **VM OS Kernel**: linux-image-6.16.3+deb13-amd64
- **Hardware**: [Beelink EQ14](https://www.bee-link.com/products/beelink-eq14-n150)
  - This is a unique product as the power supply is internal to the device. Most mini PCs have an external power brick, something I do not like. I have been running my home automation platform on one of these for the past 6+ months and have had 0 issues. 
- **CPU**: Intel N150 Alder Lake-N Graphics (Intel HD Graphics)
- **RAM**: 16 gb
- **Frigate Version**: 0.16.1
- **Camera**: USB AUKEY camera (v4l2 MJPEG input at 1280x720. The camera specs claim 1080p but green banding on the video feed lead me to believe otherwise)


## Prerequisites

### 1. BIOS Configuration (Required for GPU Passthrough)
Configure BIOS settings to enable Intel GPU hardware acceleration:

**Intel CPU/Motherboard BIOS Settings:**
```
Advanced Settings:
├── CPU Configuration
│   ├── Intel VT-d (Virtualization Technology for Directed I/O): ENABLED
│   └── Intel VT-x (Virtualization Technology): ENABLED
├── PCIe Configuration
│   ├── SR-IOV Support: ENABLED
│   └── ARI (Alternative Routing-ID Interpretation): ENABLED
└── Integrated Graphics
    ├── Primary Video Adapter: IGFX (or Auto)
    ├── Internal Graphics: ENABLED
    └── DVMT Pre-Allocated: 64MB (or higher)
```

### 2. VM Configuration in Proxmox:

** Using Proxmox CLI **
```bash
# On Proxmox host, find the Intel GPU PCI device ID
lspci | grep -i intel | grep -i vga
# Example output: 00:02.0 VGA compatible controller: Intel Corporation Alder Lake-N [UHD Graphics]

# Get detailed PCI device information
lspci -n -s 00:02.0
# Example output: 00:02.0 0300: 8086:46d1 (rev 04)
# Device ID format: vendor:device = 8086:46d1

# Add GPU to VM using qm command (replace VMID with your VM ID)
qm set <VMID> -hostpci0 00:02.0,pcie=1,rombar=1,x-vga=1

# Alternative: Add without VGA passthrough for containerized use
qm set <VMID> -hostpci0 00:02.0,pcie=1,rombar=1

# Verify the configuration
qm config <VMID> | grep hostpci
```
### 3. Verify GPU access in Debian 13 VM

**Verify GPU Passthrough in VM:**
```bash
# After VM reboot, verify GPU is accessible
lspci | grep -i intel
# Should show the Intel GPU device

# Check if /dev/dri devices are created
ls -la /dev/dri/
# Should show card0 and renderD128
```

### 2. VM OS Kernel Upgrade (Critical for Intel GPU Support)
Upgrade to the latest kernel for proper Intel GPU driver support:

```bash
# Update package lists
sudo apt update

# Install the latest kernel
sudo apt install linux-image-6.16.3+deb13-amd64 linux-headers-6.16.3+deb13-amd64

# Update GRUB and reboot
sudo update-grub
sudo reboot

# Verify kernel version after reboot
uname -r
# Expected output: 6.16.3+deb13-amd64
```

### 3. Intel GPU Drivers and Firmware
Install required Intel GPU drivers and firmware:

```bash
# Install Intel GPU drivers
sudo apt update
sudo apt install intel-gpu-tools intel-media-va-driver vainfo

# Install Intel GPU firmware (if needed)
sudo apt install firmware-misc-nonfree

# Verify GPU is detected
lspci | grep VGA
# Expected output: Intel Corporation Alder Lake-N [UHD Graphics]
```
### 4. Docker


### 5. User Permissions
Add user to required groups for GPU and video device access:

```bash
# Add user to video and render groups
sudo usermod -a -G video,render $USER

# Verify group membership
groups $USER
# Should include: video render
```

### 6. Verify GPU Access
Test that GPU acceleration is working:

```bash
# Test VAAPI functionality
vainfo
# Should show available VA-API profiles and entrypoints

# Check GPU devices
ls -la /dev/dri/
# Should show: card0 (main GPU) and renderD128 (render device)
```

## Docker Configuration

### Docker Compose Setup

The key configuration elements in `docker-compose.yml`:

```yaml
services:
  frigate:
    container_name: frigate
    privileged: true  # Required for Intel GPU access
    image: ghcr.io/blakeblackshear/frigate:stable
    shm_size: "128mb"  # Increased shared memory
    devices:
      - /dev/dri:/dev/dri  # Intel GPU device mapping
      - /dev/video0:/dev/video0  # USB camera
    volumes:
      - ./config:/config
      - ./storage:/media/frigate
      - ./models:/models  # YOLOv9 model files
      - ./labelmap:/labelmap  # COCO labels
      - /sys/devices:/sys/devices:ro  # For GPU telemetry (attempted)
      - /sys/class/drm:/sys/class/drm:ro  # Additional GPU interface
    cap_add:
      - SYS_ADMIN  # Additional capability for GPU operations
    environment:
      FRIGATE_RTSP_PASSWORD: "password"
```

## Frigate Configuration

### Key Configuration Elements

#### Detector Configuration (OpenVINO with Intel GPU)
```yaml
detectors:
  ov:
    type: openvino
    device: GPU  # Use Intel GPU for AI detection
```

#### Model Configuration (YOLOv9)
```yaml
model:
  model_type: yolo-generic
  width: 320
  height: 320
  input_tensor: nchw
  input_dtype: float
  path: /models/yolov9-t-320.onnx
  labelmap_path: /labelmap/coco-80.txt
```

#### Camera Configuration
```yaml
cameras:
  AUKEY:
    ffmpeg:
      inputs:
        - path: /dev/video0
          input_args: -f v4l2 -input_format mjpeg
          roles:
            - record
            - detect
    detect:
      enabled: true
      height: 720
      width: 1280
      fps: 5
```

## Troubleshooting Steps Taken

### 1. Initial Container Crashes
**Problem**: Container crashed with OpenVINO GPU context initialization errors

**Solution**: 
- Separated AI detection (GPU) from video processing (CPU)
- Disabled VAAPI video acceleration for USB MJPEG cameras
- Used GPU only for AI inference, not video decoding

### 2. Model File Access Issues
**Problem**: `FileNotFoundError` for YOLOv9 model

**Solutions**:
- Used absolute paths in volume mounts instead of `~/` notation
- Moved model files into project directory structure
- Created proper labelmap file with COCO-80 class names

### 3. Configuration Validation Errors
**Problem**: Invalid `live` section configuration causing crashes

**Solution**: Removed incorrectly placed live view configuration parameters

### 4. GPU Telemetry Issues (Partial Success)
**Problem**: GPU metrics not appearing in Frigate web UI

**Attempted Solutions**:
- Mounted `/sys/devices` and `/sys/class/drm` for PMU access
- Added `SYS_ADMIN` capability
- Configured `intel_gpu_device: renderD128` in telemetry section

**Status**: GPU telemetry remains non-functional due to containerization limitations with Intel PMU interface

## Final Working Configuration

### Hardware Acceleration Status
- ✅ **Intel GPU AI Detection**: Working (OpenVINO on Intel GPU)
- ✅ **Object Detection**: Functional with YOLOv9 model
- ✅ **Bounding Boxes**: Enabled in snapshots and recordings
- ❌ **GPU Telemetry**: Not working (PMU access limitation in containers)
- ❌ **Video Decode Acceleration**: Disabled (conflicts with USB MJPEG)

### Performance Results
- **Inference Speed**: ~22ms per detection
- **Detection FPS**: Stable at configured rate
- **GPU Utilization**: Intel GPU successfully used for AI workloads
- **CPU Usage**: Reduced CPU load for detection tasks

## Key Learnings

1. **USB MJPEG Cameras**: Don't benefit from GPU video decoding and can conflict with VAAPI drivers
2. **Hybrid Approach**: Use GPU for AI detection, CPU for video processing with USB cameras
3. **Container Limitations**: Intel GPU telemetry (PMU) not accessible from privileged containers on linux-image-6.16.3+deb13-amd64


## Files Structure
```
frigate/
├── docker-compose.yml
├── config/
│   └── config.yml
├── models/
│   └── yolov9-t-320.onnx
├── labelmap/
│   └── coco-80.txt
├── storage/
└── homeassistant/
    └── configuration.yaml
```

## Verification Commands

```bash
# Check container status
docker compose ps

# Verify GPU access in container
docker compose exec frigate ls -la /dev/dri/

# Check detection performance
curl -s http://localhost:5000/api/stats | grep "inference_speed"

# Monitor logs for GPU usage
docker compose logs frigate --follow | grep -i "gpu\|openvino"
```

## Notes

- Intel GPU hardware acceleration works best for AI inference tasks
- Video decode acceleration should be avoided with USB MJPEG cameras
- GPU telemetry requires additional host-level configuration beyond standard containerization
- Proper device permissions and group membership are critical for GPU access