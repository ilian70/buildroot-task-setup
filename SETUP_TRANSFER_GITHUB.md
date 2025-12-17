# Buildroot Setup: Transfer .gz + GitHub Clone Instructions

## Overview
This guide shows how to set up a complete buildroot environment using:

1. **buildroot-transfer.zip** - Buildroot configuration and board files

2. **buildroot-task from GitHub** - Fresh clone of project source code

This approach ensures you always have the latest project code from GitHub while using pre-configured buildroot settings.

---

## Prerequisites

- Linux/Unix system with git, tar, and build tools
- ~2GB free disk space
- Internet connection to clone from GitHub and buildroot repository
- **buildroot repository must be cloned from official source (branch 2025.11.x)**

---

## Part 0: Clone Buildroot Repository

Before starting with the transfer tarball, clone the official buildroot repository:

```bash
# Clone buildroot from official repository (2025.11 branch)
git clone --depth 1 --branch 2025.11 https://github.com/buildroot/buildroot.git

# Verify the clone
cd buildroot
git status
# Should show: On branch 2025.11 or detached HEAD at 2025.11 commit
```

### After cloning buildroot:

The buildroot repository is now ready. You can then apply the transfer tarball on top of it.
---

## Part 1: Directory Setup

After cloning buildroot (Part 0), your working directory structure should be:

```bash
# Create workspace (if not already in buildroot directory)
cd ~/buildroot-workspace  # Or wherever you cloned buildroot
```

Your final structure will be:
```
~/buildroot-workspace/ (or your chosen location)
├── buildroot/              (cloned in Part 0, will be updated with transfer.tar.gz)
└── buildroot-task/         (will be cloned from GitHub in Part 4)
```

**Important:** The buildroot directory already exists from Part 0. The transfer tarball will be extracted INTO this directory to preserve your cloned repository while adding configuration files.

---

## Part 2: Extract Transfer Tarball (Into Existing Buildroot)

Download the `buildroot-transfer.zip` file and extract it:

```bash
cd ~/buildroot-workspace

# Extract the tarball (will merge with existing buildroot directory)
tar xzf /path/to/buildroot-transfer.zip

# Verify extraction
ls -la
# Should show: buildroot/ and buildroot-task/
```

### What's in the tarball:

- **buildroot/.config** - Complete build configuration
- **buildroot/configs/raspberrypi5_defconfig** - Pi5 default config
- **buildroot/board/raspberrypi/** - Board configurations
- **buildroot/package/redis-image-viewer/** - Package definition files
---

## Part 3: Remove Extracted buildroot-task, Clone From GitHub

Since you'll be using the GitHub clone instead of the extracted source:

```bash
cd ~/buildroot-workspace

# Remove the extracted buildroot-task directory from tarball
rm -rf buildroot-task/

# Clone buildroot-task from GitHub
git clone https://github.com/ilian70/buildroot-task.git buildroot-task

# Verify clone
cd buildroot-task
git branch -a
# Should show: main, buildroot-package, docker branches
git remote -v
# Should show: origin https://github.com/ilian70/buildroot-task.git
```

### Check out specific branch (optional):

```bash
cd ~/buildroot-workspace/buildroot-task

# Stay on main (default)
git checkout main

# Or switch to other branches
git checkout buildroot-package
git checkout docker
```

---

## Part 4: Verify Directory Structure

After setup, verify the structure:

```bash
cd ~/buildroot-workspace

echo "=== Directory Structure ==="
tree -L 2 -I 'build|output|.*' buildroot buildroot-task

# Or use ls for simpler view
echo "Buildroot contents:"
ls -d buildroot/*/
echo
echo "buildroot-task contents:"
ls buildroot-task/*.{cpp,h,json,txt} 2>/dev/null | head -10
```

Expected output:
```
buildroot/
├── .config
├── configs/
├── board/
├── package/
│   └── redis-image-viewer/
├── Makefile
└── [other buildroot files]

buildroot-task/
├── .git/
├── .gitignore
├── CMakeLists.txt
├── app.cpp
├── app.h
├── main.cpp
├── post-build.sh
├── buildroot-package/
└── [other source files]
```

---

## Part 5: Update buildroot Configuration Paths

The extracted `.config` contains paths. Update them to point to your GitHub clone:

### Automatic Path Update

```bash
cd ~/buildroot-workspace/buildroot

# Replace absolute paths with relative paths
sed -i 's|/home/innocore/Projects/buildroot-task|../buildroot-task|g' .config

# Verify the change
grep "BR2_ROOTFS_POST_BUILD_SCRIPT" .config
# Should show: "board/raspberrypi5/post-build.sh ../buildroot-task/post-build.sh"
```
---

## Part 6: Verify Configuration

```bash
cd ~/buildroot-workspace/buildroot

# Check critical settings
echo "=== Configuration Check ==="
echo "Package enabled:"
grep "BR2_PACKAGE_REDIS_IMAGE_VIEWER" .config

echo "Post-build script path:"
grep "BR2_ROOTFS_POST_BUILD_SCRIPT=" .config | grep -v "^#"

echo "Architecture:"
grep "BR2_aarch64" .config | head -1

echo "Board config:"
grep "BR2_DEFCONFIG\|raspberrypi" .config | grep -v "^#"
```

Expected output:
```
BR2_PACKAGE_REDIS_IMAGE_VIEWER=y
BR2_ROOTFS_POST_BUILD_SCRIPT="board/raspberrypi5/post-build.sh ../buildroot-task/post-build.sh"
BR2_aarch64=y
BR2_DEFCONFIG="/path/to/buildroot/configs/raspberrypi5_defconfig"
```

---

## Part 7: Prepare for Build

Ensure all dependencies are present:

```bash
cd ~/buildroot-workspace/buildroot-task

# Make post-build script executable
chmod +x post-build.sh

# Verify key files exist
echo "=== Source Files Check ==="
ls -lh CMakeLists.txt post-build.sh
ls -1 *.cpp | head -5
ls -1 *.h | head -5

cd ../buildroot

# Verify buildroot package
ls -la package/redis-image-viewer/
```

---

## Part 8: Build

### Full Build (Recommended for first time)

```bash
cd ~/buildroot-workspace/buildroot

# Build everything
make -j$(nproc)

# Wait 45-60 minutes for full build...
```

### Incremental Build (If code changed)

```bash
cd ~/buildroot-workspace/buildroot

# Rebuild just your package
make redis-image-viewer-rebuild

# Then build the complete image
make
```

### Clean and Rebuild

```bash
cd ~/buildroot-workspace/buildroot

# Remove build artifacts
make clean

# Rebuild everything
make -j$(nproc)
```

---

## Part 9: Verify Output

After successful build:

```bash
cd ~/buildroot-workspace/buildroot

# Check generated images
ls -lh output/images/
# Should show:
#   - sdcard.img (SD card image, ~300MB)
#   - rootfs.ext2/ext4 (Root filesystem, ~256MB)
#   - boot.vfat (Boot partition, ~32MB)
#   - Image (Linux kernel, ~24MB)

# Verify your application was built
file output/target/usr/bin/redis_image_viewer
# Should show: ELF 64-bit LSB executable
```

---

## Part 10: Deploy to Raspberry Pi

### Write to SD Card (Linux)

```bash
# Insert SD card and identify its device
lsblk
# Look for your SD card device (e.g., /dev/sdc, /dev/sdd)

# Write image (replace /dev/sdX with your device)
sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M conv=fsync
sudo sync
sudo eject /dev/sdX
```

### Boot and Test

1. Insert SD card into Raspberry Pi 5
2. Power on and wait for boot
3. Connect via SSH: `ssh root@<pi-ip>` (password: root)
4. Check your application:
   ```bash
   systemctl status redis-image-viewer
   # Should show: active (running)
   ```

---


## Troubleshooting

### Build Error: "Cannot find post-build.sh"

**Cause:** Path in .config is incorrect

**Solution:**
```bash
cd buildroot

# Verify path is correct
ls -la ../buildroot-task/post-build.sh

# If not found, update .config:
sed -i 's|..*/buildroot-task|../buildroot-task|g' .config
make menuconfig  # Verify paths
```

### Build Error: "Package not found"

**Cause:** redis-image-viewer package not properly installed

**Solution:**
```bash
cd buildroot

# Verify package exists
ls -la package/redis-image-viewer/

# If missing, re-extract from tarball
# Or copy from GitHub clone:
cp -r buildroot-task/buildroot-package/redis-image-viewer \
      package/redis-image-viewer
```

### Build Error: "CMake not found"

**Cause:** Missing dependencies

**Solution:**
```bash
# Install required build tools
sudo apt-get install build-essential cmake git subversion \
    libncurses5-dev libc6-dev libssl-dev unzip wget bc

# Retry build
cd buildroot
make clean
make -j$(nproc)
```

### Git Clone Fails

**Cause:** Network or permissions issue

**Solution:**
```bash
# Try HTTPS again
git clone https://github.com/ilian70/buildroot-task.git

# Or use SSH if you have keys configured
git clone git@github.com:ilian70/buildroot-task.git
```

---

## Quick Reference Commands

```bash
# Navigate to buildroot
cd ~/buildroot-workspace/buildroot

# Configure
make menuconfig

# Build
make -j$(nproc)                    # Full build
make redis-image-viewer-rebuild    # Rebuild package only
make clean                          # Clean all

# Check status
make redis-image-viewer            # Build just package
cat output/build/redis-image-viewer-1.0/.log  # View build log

```

---

## Part Complete Setup One-Liner

If you want to automate the entire setup:

```bash
#!/bin/bash
WORKSPACE=~/buildroot-workspace
mkdir -p "$WORKSPACE"
cd "$WORKSPACE"

# Extract transfer tarball
tar xzf /path/to/buildroot-transfer.zip

# Remove extracted source, clone from GitHub instead
rm -rf buildroot-task
git clone https://github.com/ilian70/buildroot-task.git buildroot-task

# Update paths
cd buildroot
sed -i 's|/home/.*buildroot-task|../buildroot-task|g' .config

# Verify
echo "✓ Setup complete!"
echo "  Location: $WORKSPACE"
echo "  Ready to build: make -j\$(nproc)"
```

---

## File Checklist

Before building, verify these files exist:

```
✓ buildroot/.config
✓ buildroot/configs/raspberrypi5_defconfig
✓ buildroot/board/raspberrypi/
✓ buildroot/package/redis-image-viewer/Config.in
✓ buildroot/package/redis-image-viewer/redis-image-viewer.mk
✓ buildroot/package/redis-image-viewer/redis-image-viewer.service
✓ buildroot-task/.git/
✓ buildroot-task/CMakeLists.txt
✓ buildroot-task/main.cpp
✓ buildroot-task/app.cpp
✓ buildroot-task/post-build.sh (executable)
✓ buildroot-task/app.cfg.json
```

---
Created: December 16, 2025  
For: redis-image-viewer buildroot project  
Target: Raspberry Pi 5
