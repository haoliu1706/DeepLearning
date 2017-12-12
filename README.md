This repository will contain mostly documentation about my Tensorflow (CPU + GPU) setup incl. the prerequirements on (L)Ubuntu Linux.

# My Hardware Setup #
- Board: Asrock Fatal1ty X370 Gaming-ITX/ac (Bios 3.40)
- CPU: AMD A12-9800E APU (4 CPU cores and 8 GPU cores [= 512 stream processors])
- CPU Cooler: Noctua NH-L9a incl. AM4 retention kit
- RAM: 2x 8 GB DDR4 2400 Kingston HyperX
- Video Card: integrated in CPU, currently only 512 MB set as shared video memory due to the Bios but this will change with new 4.xx Bios versions, then up to 2048 MB shared video memory is usable
- Storage: 256 GB Samsung 950 Pro M.2 SSD
- Power Supply: Corsair SF450 SFX
- Case: Cooltek U1

# My Basic Software Setup #
- pure UEFI install
- Lubuntu 17.10
- Kernel: the current version of the 4.14.x branch (via Ubuntu Mainline Kernels) and the official 4.13 Ubuntu 17.10 version as fallback
- Filesystem: DM-LUKS encrypted SSD with BTRFS filesystem
- Desktop Environment: LXDE

# AMD OpenCL Setup #
The full install of the AMDGPU Pro driver (current version is 17.40) is not supported and should not work on my setup so i use a hybrid approach. This is not supported by AMD but fortunatly it works.

I use the existing Arch Linux [https://aur.archlinux.org/packages/opencl-amd/] work as basis and adapted it for my needs.

To get OpenCL running we need to extract the OpenCL related parts from the AMDGPU Pro driver.

- install "clinfo" to install the prerequirements, clinfo itself is used to check if the OpenCL libraries are working
```
sudo apt install clinfo
```

I do all the downloading and extracting stuff in the "$HOME/Downloads" directory. Change it to your needs if you want.

- download the current AMDGPU-PRO driver: http://support.amd.com/en-us/kb-articles/Pages/AMDGPU-PRO-Driver-for-Linux-Release-Notes.aspx
- open a shell, i use LXTerminal (Bash)
- change the directory to "$HOME/Downloads"
```
cd $HOME/Downloads
```

ATTENTION: I hard coded the "$HOME/Downloads" directory in the script. If you want to use another directory then you have to change it to your needs. I know there are better / more simple approaches to extract the parts but the following is working for me.

The "/etc/OpenCL/vendors" path is mandatory as far as i know but you can change the "/opt/amdgpu-pro/" path to another one if you want. Then you have to change the commands/script accordingly.

- create the script that will do the extraction of the OpenCL parts
```
vi extract_amdgpu-pro_opencl.sh
```
- and add the script content
```
#!/bin/bash

# change version (e.g. 17.40-492261) as needed
# you have to extract the AMDGPU-PRO driver if you want to know the used libdrm version
MAJORVERSION="17.40"
MINORVERSION="492261"
LIBDRMVERSION="2.4.82"

# create the needed directories
mkdir -p $HOME/Downloads/temp/icd
mkdir -p $HOME/Downloads/temp/libdrm
mkdir -p $HOME/Downloads/root_directory/etc/OpenCL/vendors
mkdir -p $HOME/Downloads/root_directory/opt/amdgpu-pro/lib64
mkdir -p $HOME/Downloads/root_directory/opt/amdgpu-pro/share/libdrm/

# extract the OpenCL parts from the AMDGPU Pro driver
cd $HOME/Downloads
tar xJf amdgpu-pro-${MAJORVERSION}-${MINORVERSION}.tar.xz
cp $HOME/Downloads/amdgpu-pro-${MAJORVERSION}-${MINORVERSION}/opencl-amdgpu-pro-icd_${MAJORVERSION}-${MINORVERSION}_amd64.deb $HOME/Downloads/temp/icd
cp $HOME/Downloads/amdgpu-pro-${MAJORVERSION}-${MINORVERSION}/libdrm-amdgpu-pro-amdgpu1_${LIBDRMVERSION}-${MINORVERSION}_amd64.deb $HOME/Downloads/temp/libdrm

# extract the icd parts
cd $HOME/Downloads/temp/icd
ar x opencl-amdgpu-pro-icd_${MAJORVERSION}-${MINORVERSION}_amd64.deb
tar xJf data.tar.xz
cp $HOME/Downloads/temp/icd/etc/OpenCL/vendors/amdocl64.icd $HOME/Downloads/root_directory/etc/OpenCL/vendors
cp $HOME/Downloads/temp/icd/opt/amdgpu-pro/lib/x86_64-linux-gnu/* $HOME/Downloads/root_directory/opt/amdgpu-pro/lib64

# extract the libdrm parts
cd $HOME/Downloads/temp/libdrm
ar x libdrm-amdgpu-pro-amdgpu1_${LIBDRMVERSION}-${MINORVERSION}_amd64.deb
tar xJf data.tar.xz
cp $HOME/Downloads/temp/libdrm/opt/amdgpu-pro/lib/x86_64-linux-gnu/* $HOME/Downloads/root_directory/opt/amdgpu-pro/lib64

# add the amdgpu.ids file
cp /usr/share/libdrm/amdgpu.ids $HOME/Downloads/root_directory/opt/amdgpu-pro/share/libdrm

# clean up
rm -r $HOME/Downloads/temp
rm -r $HOME/Downloads/amdgpu-pro-${MAJORVERSION}-${MINORVERSION}
```
- make the script executable
```
chmod u+x extract_amdgpu-pro_opencl.sh
```
- run the script
```
./extract_amdgpu-pro_opencl.sh
```
- copy the extracted OpenCL parts to their final places
```
sudo cp -ri $HOME/Downloads/root_directory/etc/* /etc
sudo cp -ri $HOME/Downloads/root_directory/opt/* /opt
```
- an OpenCL application will only work if you provide it the location of the OpenCL libraries, "LD_LIBRARY_PATH=/opt/amdgpu-pro/lib64" will do this
- you can also use the available options (profile, bashrc, ...) in a Linux distribution to make the OpenCL libraries known to all users in the system
- check if it works
```
LD_LIBRARY_PATH=/opt/amdgpu-pro/lib64 clinfo
```
- the result should something like this, it depends on the used GPU and driver version
```
steven@box:~$ LD_LIBRARY_PATH=/opt/amdgpu-pro/lib64 clinfo
Number of platforms                               1
  Platform Name                                   AMD Accelerated Parallel Processing
  Platform Vendor                                 Advanced Micro Devices, Inc.
  Platform Version                                OpenCL 2.0 AMD-APP (2482.3)
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_icd cl_amd_event_callback cl_amd_offline_devices 
  Platform Extensions function suffix             AMD

  Platform Name                                   AMD Accelerated Parallel Processing
Number of devices                                 1
  Device Name                                     Carrizo
  Device Vendor                                   Advanced Micro Devices, Inc.
  Device Vendor ID                                0x1002
  Device Version                                  OpenCL 1.2 AMD-APP (2482.3)
  Driver Version                                  2482.3
  Device OpenCL C Version                         OpenCL C 1.2 
  Device Type                                     GPU
  Device Profile                                  FULL_PROFILE
  Device Board Name (AMD)                         AMD Radeon Graphics
  Device Topology (AMD)                           PCI-E, 00:01.0
  Max compute units                               8
  SIMD per compute unit (AMD)                     4
  SIMD width (AMD)                                16
  SIMD instruction width (AMD)                    1
  Max clock frequency                             900MHz
  Graphics IP (AMD)                               8.0
  Device Partition                                (core)
    Max number of sub-devices                     8
    Supported partition types                     none specified
  Max work item dimensions                        3
  Max work item sizes                             256x256x256
  Max work group size                             256
  Preferred work group size multiple              64
  Wavefront width (AMD)                           64
  Preferred / native vector sizes                 
    char                                                 4 / 4       
    short                                                2 / 2       
    int                                                  1 / 1       
    long                                                 1 / 1       
    half                                                 1 / 1        (cl_khr_fp16)
    float                                                1 / 1       
    double                                               1 / 1        (cl_khr_fp64)
  Half-precision Floating-point support           (cl_khr_fp16)
    Denormals                                     No
    Infinity and NANs                             No
    Round to nearest                              No
    Round to zero                                 No
    Round to infinity                             No
    IEEE754-2008 fused multiply-add               No
    Support is emulated in software               No
    Correctly-rounded divide and sqrt operations  No
  Single-precision Floating-point support         (core)
    Denormals                                     No
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             Yes
    IEEE754-2008 fused multiply-add               Yes
    Support is emulated in software               No
    Correctly-rounded divide and sqrt operations  Yes
  Double-precision Floating-point support         (cl_khr_fp64)
    Denormals                                     Yes
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             Yes
    IEEE754-2008 fused multiply-add               Yes
    Support is emulated in software               No
    Correctly-rounded divide and sqrt operations  No
  Address bits                                    64, Little-Endian
  Global memory size                              2036277248 (1.896GiB)
  Global free memory (AMD)                        1163236 (1.109GiB)
  Global memory channels (AMD)                    4
  Global memory banks per channel (AMD)           8
  Global memory bank width (AMD)                  256 bytes
  Error Correction support                        No
  Max memory allocation                           1369020825 (1.275GiB)
  Unified memory for Host and Device              Yes
  Minimum alignment for any data type             128 bytes
  Alignment of base address                       2048 bits (256 bytes)
  Global Memory cache type                        Read/Write
  Global Memory cache size                        16384
  Global Memory cache line                        64 bytes
  Image support                                   Yes
    Max number of samplers per kernel             16
    Max size for 1D images from buffer            134217728 pixels
    Max 1D or 2D image array size                 2048 images
    Base address alignment for 2D image buffers   256 bytes
    Pitch alignment for 2D image buffers          256 bytes
    Max 2D image size                             16384x16384 pixels
    Max 3D image size                             2048x2048x2048 pixels
    Max number of read image args                 128
    Max number of write image args                8
  Local memory type                               Local
  Local memory size                               32768 (32KiB)
  Local memory syze per CU (AMD)                  65536 (64KiB)
  Local memory banks (AMD)                        32
  Max constant buffer size                        1369020825 (1.275GiB)
  Max number of constant args                     8
  Max size of kernel argument                     1024
  Queue properties                                
    Out-of-order execution                        No
    Profiling                                     Yes
  Prefer user sync for interop                    Yes
  Profiling timer resolution                      1ns
  Profiling timer offset since Epoch (AMD)        1513061695487142552ns (Tue Dec 12 07:54:55 2017)
  Execution capabilities                          
    Run OpenCL kernels                            Yes
    Run native kernels                            No
    Thread trace supported (AMD)                  Yes
    SPIR versions                                 1.2
  printf() buffer size                            1048576 (1024KiB)
  Built-in kernels                                
  Device Available                                Yes
  Compiler Available                              Yes
  Linker Available                                Yes
  Device Extensions                               cl_khr_fp64 cl_amd_fp64 cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_int64_base_atomics cl_khr_int64_extended_atomics cl_khr_3d_image_writes cl_khr_byte_addressable_store cl_khr_fp16 cl_khr_gl_sharing cl_amd_device_attribute_query cl_amd_vec3 cl_amd_printf cl_amd_media_ops cl_amd_media_ops2 cl_amd_popcnt cl_khr_image2d_from_buffer cl_khr_spir cl_khr_gl_event 

NULL platform behavior
  clGetPlatformInfo(NULL, CL_PLATFORM_NAME, ...)  AMD Accelerated Parallel Processing
  clGetDeviceIDs(NULL, CL_DEVICE_TYPE_ALL, ...)   Success [AMD]
  clCreateContext(NULL, ...) [default]            Success [AMD]
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_CPU)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_GPU)  Success (1)
    Platform Name                                 AMD Accelerated Parallel Processing
    Device Name                                   Carrizo
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_ACCELERATOR)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_CUSTOM)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_ALL)  Success (1)
    Platform Name                                 AMD Accelerated Parallel Processing
    Device Name                                   Carrizo

ICD loader properties
  ICD loader Name                                 OpenCL ICD Loader
  ICD loader Vendor                               OCL Icd free software
  ICD loader Version                              2.2.11
  ICD loader Profile                              OpenCL 2.1
```
