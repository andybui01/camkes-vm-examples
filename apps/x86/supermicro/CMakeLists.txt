

#
# Copyright 2022, UNSW (ABN 57 195 873 179)
#
# SPDX-License-Identifier: BSD-2-Clause
#

cmake_minimum_required(VERSION 3.8.2)

project(supermicro)

# Include CAmkES VM helper functions
include(${CAMKES_VM_HELPERS_PATH})
find_package(camkes-vm-linux REQUIRED)
include(${CAMKES_VM_LINUX_HELPERS_PATH})
include(${CAMKES_VM_LINUX_SOURCE_HELPERS_PATH})
include(ExternalProject)
include(external-project-helpers)

ExternalProject_Add(
    virtio_vsock_app
    URL
    file://${CAMKES_VM_LINUX_DIR}/camkes-linux-artifacts/camkes-linux-apps/demo-apps/virtio_vsock
    BINARY_DIR
    ${CMAKE_CURRENT_BINARY_DIR}/virtio_vsock_app
    INSTALL_COMMAND
    ""
    BUILD_ALWAYS 1
    ON
    CMAKE_ARGS
    -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
    -DCMAKE_TOOLCHAIN_FILE=${LINUX_32BIT_TOOLCHAIN}
    -DCMAKE_C_FLAGS="-m32"
)

AddExternalProjFilesToOverlay(
    virtio_vsock_app
    ${CMAKE_CURRENT_BINARY_DIR}/virtio_vsock_app
    overlay
    "usr/bin"
    FILES
    vm_host
)

AddExternalProjFilesToOverlay(
    virtio_vsock_app
    ${CMAKE_CURRENT_BINARY_DIR}/virtio_vsock_app
    overlay
    "usr/bin"
    FILES
    vm_client
)

set(NumberVMs 4)
MATH(EXPR NumberVMsLess1 "${NumberVMs}-1")

# Declare VM component: Init
foreach(i RANGE ${NumberVMsLess1})
    file(GLOB "init${i}_extra" "src/cross_vm_connections_vm${i}.c")
    DeclareCAmkESVM("Init${i}" EXTRA_SOURCES "${init${i}_extra}")
endforeach()

DeclareCamkesComponent(virtqueueinit)

# Get 32-bit Linux VM files
GetSupermicroLinuxKernelFile(kernel_file)
GetArchDefaultLinuxRootfsFile("32" rootfs_file)

# Decompress Linux Kernel image
DecompressLinuxKernel(extract_linux_kernel decompressed_kernel ${kernel_file})

# Add Linux VM images into file server
AddOverlayDirToRootfs(
    overlay
    ${rootfs_file}
    "buildroot"
    "rootfs_install"
    output_overlayed_rootfs_location
    rootfs_target
    GZIP
    DEPENDS
    virtio_vsock_app
)
AddToFileServer("bzimage" ${decompressed_kernel} DEPENDS extract_linux_kernel)
AddToFileServer("rootfs.cpio" ${output_overlayed_rootfs_location} DEPENDS rootfs_target)

# Initialise CAmkES Root Server with addition CPP includes
DeclareCAmkESVMRootServer(supermicro.camkes CPP_INCLUDES configurations)

if(SIMULATION)
    include(simulation)
    GenerateSimulateScript()
endif()
