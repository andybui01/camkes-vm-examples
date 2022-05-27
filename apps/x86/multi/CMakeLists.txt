#
# Copyright 2022, UNSW (ABN 57 195 873 179)
#
# SPDX-License-Identifier: BSD-2-Clause
#

cmake_minimum_required(VERSION 3.8.2)

project(multi)

# Include CAmkES VM helper functions
include(${CAMKES_VM_HELPERS_PATH})
find_package(camkes-vm-linux REQUIRED)
include(${CAMKES_VM_LINUX_HELPERS_PATH})

set(NumberVMs 3)
MATH(EXPR NumberVMsLess1 "${NumberVMs}-1")

# Retrieve Init0 extra src files
file(GLOB init0_extra src/cross_vm_connections_vm0.c)
file(GLOB init1_extra src/cross_vm_connections_vm1.c)

# Declare VM component: Init
DeclareCAmkESVM(Init0 EXTRA_SOURCES ${init0_extra})
DeclareCAmkESVM(Init1 EXTRA_SOURCES ${init1_extra})
DeclareCAmkESVM(Init2)

DeclareCamkesComponent(virtqueueinit)

# Get 32-bit Linux VM files
GetArchDefaultLinuxKernelFile("32" kernel_file)
GetArchDefaultLinuxRootfsFile("32" rootfs_file)

# Decompress Linux Kernel image
DecompressLinuxKernel(extract_linux_kernel decompressed_kernel ${kernel_file})

# Add Linux VM images into file server
AddToFileServer("bzimage" ${decompressed_kernel} DEPENDS extract_linux_kernel)
AddToFileServer("rootfs.cpio" ${rootfs_file})

# Initialise CAmkES Root Server with addition CPP includes
DeclareCAmkESVMRootServer(multi.camkes CPP_INCLUDES configurations)
