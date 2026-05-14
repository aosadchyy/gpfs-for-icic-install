---
layout: default
title: IBM Spectrum Scale (GPFS) Installation Guide for IBM Cloud Infrastructure Center
description: Step-by-step guide for installing and configuring IBM Spectrum Scale (GPFS) for IBM Cloud Infrastructure Center
---

# IBM Spectrum Scale (GPFS) Installation Guide for IBM Cloud Infrastructure Center

This guide provides instructions for installing IBM Spectrum Scale (formerly GPFS) version 5.1.9.2 for use with IBM Cloud Infrastructure Center (ICIC).

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Important Configuration Notes](#important-configuration-notes)
- [Installation Steps](#installation-steps)
- [Cluster Configuration](#cluster-configuration)
- [Verification and Monitoring](#verification-and-monitoring)
- [Troubleshooting](#troubleshooting)

## Overview

IBM Spectrum Scale must be mounted on both Management (MGMT) and Compute (COMP) nodes to ensure proper functionality, particularly for Boot from Volume (BFV) operations with Glance images.

### Mount Points
- **Management Node**: `/var/lib/glance/images`
- **Compute Node**: `/gpfs/icic_gpfs/icic/images`

> **Note**: If GPFS is not mounted on both nodes, Boot from Volume operations will fail. Images must be replicated from MGMT to COMP nodes.

## Prerequisites

- Red Hat Enterprise Linux 8 or 9
- IBM Spectrum Scale installation media (e.g. 5.1.9.2 or later)
- Root access to all nodes
- Network connectivity between nodes

## Important Configuration Notes

### 1. Libvirt Configuration

To avoid SELinux errors, disable the `remember_owner` setting in libvirt:

```bash
# Edit /etc/libvirt/qemu.conf
remember_owner=0

# Restart libvirtd service
systemctl restart libvirtd
```

### 2. Kernel Parameters (RHEL 8/9)

For Red Hat Enterprise Linux 8 or 9, add the `vmalloc` parameter to all boot entries:

```bash
# Add vmalloc parameter to all kernel boot entries
grubby --update-kernel=ALL --args="vmalloc=4096G"

# Run zipl command
zipl -V

# Reboot the node
reboot
```

## Installation Steps

### Step 1: Import GPG Keys

```bash
# Navigate to the public keys directory
cd /usr/lpp/mmfs/5.1.9.2/Public_Keys/

# Import the Red Hat GPG key
rpm --import RPM-GPG-KEY-redhat-release
```

### Step 2: Install Required Packages

```bash
# Navigate to the installation directory
cd /usr/lpp/mmfs/5.1.9.2/

# Install librdkafka package (adjust architecture as needed)
yum install /usr/lpp/mmfs/5.1.9.2/gpfs_rpms/rhel8/gpfs.librdkafka-5.1.9-2.el8.s390x.rpm

# Locate and install mmcrcluster
find / -name mmcrcluster
yum install mmcrcluster
```

### Step 3: Set Environment Path

```bash
# Add GPFS binaries to PATH
export PATH="/usr/lpp/mmfs/bin:$PATH"

# Add to ~/.bashrc for persistence
echo 'export PATH="/usr/lpp/mmfs/bin:$PATH"' >> ~/.bashrc
```

## Cluster Configuration

### Create GPFS Cluster

```bash
# Create cluster with manager-quorum role
# Replace <compute-node> with your compute node hostname
mmcrcluster -N <compute-node>:manager-quorum

# Accept server license
# Replace <compute-node> with your compute node hostname
/usr/lpp/mmfs/bin/mmchlicense server --accept -N <compute-node>
```

### Create Network Shared Disks (NSD)

```bash
# Create NSD using stanza file
mmcrnsd -F stanzaFile -v no
```

## Verification and Monitoring

### Check Cluster Status

```bash
# Check node state
mmgetstate -aLs

# List cluster information
mmlscluster

# List all nodes
mmlsnode

# List network shared disks
mmlsnsd

# Verify network connectivity
mmnetverify
```

### Check Filesystem Status

```bash
# List all filesystems
mmlsfs all

# List all mount points
mmlsmount all
```

### Check Services

```bash
# Check GPFS service status
systemctl status gpfs

# Check mmautoload service status
systemctl status mmautoload
```

### Access Web Interface

Access the GPFS management interface at:
```
https://<management-node-ip>/gui#services-view-GPFS/1
```

## Troubleshooting

### GPFS Service Down

If GPFS goes down, follow these diagnostic steps:

#### Diagnose the Issue

```bash
# Check node state
mmgetstate -aLs

# Check cluster configuration
mmlscluster

# Check service status
systemctl status gpfs
systemctl status mmautoload

# Check recent logs
tail -10 /var/adm/ras/mmfs.log.latest
```

#### Enable and Restart Services

```bash
# Enable GPFS service
systemctl enable gpfs

# Start GPFS on all nodes
mmstartup -a
```

### Kernel Module Issues

If you encounter kernel module issues after a kernel update, you may need to recompile the GPFS kernel modules:

```bash
# Recompile kernel modules
mmbuildgpl

# Check module permissions
ls -la /lib/modules/$(uname -r)/extra/

# Fix permissions if needed
chmod u+x /lib/modules/$(uname -r)/extra/*
```

The following kernel modules should be present:
- `mmfs26.ko`
- `mmfslinux.ko`
- `tracedev.ko`

### Using the Event Log

For automated troubleshooting, access the GPFS event log through the web interface:

1. Navigate to: `https://<management-node-ip>/gui#services-view-GPFS/1`
2. Review the event log
3. Click on "Start Fix Procedure" for automated remediation

## Additional Resources

- [IBM Spectrum Scale Administration Guide](https://www.ibm.com/docs/en/STXKQY_5.1.9/pdf/scale_adm.pdf)
- [GPFS Cluster Installation Tutorial](https://www.cnaf.infn.it/~vladimir/corso_gpfs/Install%20and%20configure%20a%20GPFS%20cluster)

---

*Last Updated: {{ site.time | date: '%B %d, %Y' }}*
