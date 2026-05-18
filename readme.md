# V100 Cluster

## Devices

### PDU
http://10.128.0.62/

### headnode


https://10.128.0.163/

ipmitool  -H 10.128.0.163 -U ADMIN -P ADMIN chassis power on


## RKE2

### Control Plane

> TODO

### Worker

On the control plan node get the join token:
```bash
cat /var/lib/rancher/rke2/server/node-token
```

On the worker

```
mkdir -p ./rke2
pushd rke2
  curl -sfL https://get.rke2.io --output install.sh
  chmod +x ./install.sh 
popd

 TOKEN=FOOBAR
sudo mkdir -p /etc/rancher/rke2/
cat << EOF | sudo tee /etc/rancher/rke2/config.yaml
server: https://10.128.0.200:9345
token: $TOKEN
node-taint:
- nvidia.com/gpu=present:NoSchedule
EOF

cat << EOF | sudo tee /etc/rancher/rke2/registries.yaml
mirrors:
  nvcr.io:
    endpoint:
    - http://10.128.0.200:8050/v2/nvcr.io
  docker.io:
    endpoint:
    - http://10.128.0.200:8050/v2/docker.io
  quay.io:
    endpoint:
    - http://10.128.0.200:8050/v2/docker.io
  registry.k8s.io:
    endpoint:
    - http://10.128.0.200:8050/v2/registry.k8s.io
  registry.local:
    endpoint:
    - http://10.128.0.200:8050/v2/
  registry5000.local:
    endpoint:
    - http://10.128.0.200:5000/v2/
EOF


sudo \
INSTALL_RKE2_VERSION=v1.35.3+rke2r3 \
INSTALL_RKE2_TYPE=agent \
./rke2/install.sh

sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
```


## Hardware config

### SMC
#### BMC
1. Reset password
```bash
wget https://www.supermicro.com/wdl/utility/IPMICFG/IPMICFG_1.35.1_build.230912.zip

mkdir IPMICFG
cd IPMICFG/
unzip ../IPMICFG_1.35.1_build.230912.zip
cd IPMICFG_1.35.1_build.230912/Linux/64bit/

sudo ./IPMICFG-Linux.x86_64 -user setpwd 2 initial0
```

```
ipmitool lan set 1 ipsrc static
ipmitool lan set 1 netmask 255.255.255.0
ipmitool lan set 1 ipaddr 10.128.0.24
ipmitool lan set 1 defgw ipaddr 10.128.0.1

```

#### Firmware update

BMC 
```bash
ryan@hn01:~/firmware/smc/X11SSL-F/X11SSL-F_3.6_AS1.79_SUM2.14.0$ ./sum_2.14.0_Linux_x86_64_20240215/sum_2.14.0_Linux_x86_64/sum -i 10.128.0.21 -u ADMIN -p initial0 -c UpdateBmc --file ./BMC_X11AST2400-3101MS_20251128_1.79_STDsp/BMC_X11AST2400-3101MS_20251117_1.79_STDsp/BMC_X11AST2400-3101MS_20251128_1.79_STDsp.bin


./sum_2.14.0_Linux_x86_64_20240215/sum_2.14.0_Linux_x86_64/sum -i 10.128.0.23 -u ADMIN -p initial0 -c UpdateBmc --file ./BMC_X11AST2400-3101MS_20251128_1.79_STDsp/BMC_X11AST2400-3101MS_20251117_1.79_STDsp/BMC_X11AST2400-3101MS_20251128_1.79_STDsp.bin
```

BIOS
```bash
./sum_2.14.0_Linux_x86_64_20240215/sum_2.14.0_Linux_x86_64/sum -i 10.128.0.24 -u ADMIN -p initial0 -c UpdateBios --file ./BIOS_X11SSLF-089A_20250425_3.6_STD/BIOS_X11SSLF-089A_20250425_3.6_STD.bin

```

### IB Cards
> https://forums.servethehome.com/index.php?threads/lenovo-connectx-4-card-flash-to-mellanox.32434/#post-369048

#### Take 1 (do not follow)
Add the doca repo and install mst

```bash
sudo su
export DOCA_URL="https://linux.mellanox.com/public/repo/doca/3.3.0/ubuntu24.04/x86_64/"
curl https://linux.mellanox.com/public/repo/doca/GPG-KEY-Mellanox.pub | gpg --dearmor > /etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub
echo "deb [signed-by=/etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub] $DOCA_URL ./" > /etc/apt/sources.list.d/doca.list
sudo apt-get update


sudo apt install mft mstflint
```

```
ryan@quad4:~$ lspci | grep ConnectX-4
06:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
07:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]

ryan@quad4:~$ sudo flint -d 0000:06:00.0 query full
Image type:            FS3
FW Version:            12.27.6008
FW Release Date:       29.5.2020
Part Number:           06W1HY_0JJN39_Ax
Description:           Mellanox ConnectX-4 Single Port EDR PCIE Adapter LP
Product Version:       12.27.6008
Rom Info:              type=PXE version=3.5.903 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             506b4b0300085b7e        4
Base MAC:              506b4b085b7e            4
Image VSD:             N/A
Device VSD:            N/A
PSID:                  DEL2180110032
Security Attributes:   N/A
Default Update Method: fw_ctrl

ryan@quad4:~$ sudo flint -d 0000:07:00.0 query full
Image type:            FS3
FW Version:            12.25.1020
FW Release Date:       30.4.2019
Part Number:           06W1HY_0JJN39_Ax
Description:           Mellanox ConnectX-4 Single Port EDR PCIE Adapter LP
Product Version:       12.25.1020
Rom Info:              type=PXE version=3.5.701 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             ec0d9a030092a51a        4
Base MAC:              ec0d9a92a51a            4
Image VSD:             N/A
Device VSD:            N/A
PSID:                  DEL2180110032
Security Attributes:   N/A
Default Update Method: fw_ctrl
```


1. Get Firmware
```
mkdir ~/firmware -p
pushd ~/firmware
    wget https://www.mellanox.com/downloads/firmware/fw-ConnectX4-rel-12_28_2302-MCX455A-ECA_Ax-UEFI-14.21.17-FlexBoot-3.6.102.bin.zip

    unzip fw-ConnectX4-rel-12_28_2302-MCX455A-ECA_Ax-UEFI-14.21.17-FlexBoot-3.6.102.bin.zip
popd
```

1. Try to flash it and it should error with PSID is different

```
ryan@quad4:~/firmware$ sudo mstflint -d 0000:06:00.0 -i ~/firmware/fw-ConnectX4-rel-12_28_2302-MCX455A-ECA_Ax-UEFI-14.21.17-FlexBoot-3.6.102.bin burn

    Current FW version on flash:  12.27.6008
    New FW version:               12.28.2302


-E- PSID mismatch. The PSID on flash (DEL2180110032) differs from the PSID in the given image (MT_2180110032).
``` 

2. Flash
```

```


#### Take 2

> Use this one

1. Install OEM MFT
> https://forums.servethehome.com/index.php?threads/trying-to-flash-fb-ocp2-0-mellanox-connect-x-4-lx-50g-to-mellanox-firmware-no-mfg_info.48375/

```bash
sudo apt-get install -y gcc make dkms unzip

mkdir -p firmware/mellanox/cx4
pushd firmware/mellanox/cx4
  wget https://www.mellanox.com/downloads/MFT/mft-4.35.0-159-x86_64-deb.tgz
  tar -xvf mft-4.35.0-159-x86_64-deb.tgz

  sudo ./mft-4.35.0-159-x86_64-deb/install.sh --oem

  wget https://www.mellanox.com/downloads/firmware/fw-ConnectX4-rel-12_28_2302-MCX455A-ECA_Ax-UEFI-14.21.17-FlexBoot-3.6.102.bin.zip

  unzip fw-ConnectX4-rel-12_28_2302-MCX455A-ECA_Ax-UEFI-14.21.17-FlexBoot-3.6.102.bin.zip

popd
```

1. Capture adapter info

   1. quad04
     ```bash
     ryan@quad4:~$ lspci | grep ConnectX-4
     06:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
     07:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]

     ryan@quad4:~$ sudo flint -d 0000:06:00.0 query full
     Image type:            FS3
     FW Version:            12.27.6008
     FW Release Date:       29.5.2020
     Part Number:           06W1HY_0JJN39_Ax
     Description:           Mellanox ConnectX-4 Single Port EDR PCIE Adapter LP
     Product Version:       12.27.6008
     Rom Info:              type=PXE version=3.5.903 cpu=AMD64
     Description:           UID                GuidsNumber
     Base GUID:             506b4b0300085b7e        4
     Base MAC:              506b4b085b7e            4
     Image VSD:             N/A
     Device VSD:            N/A
     PSID:                  DEL2180110032
     Security Attributes:   N/A
     Default Update Method: fw_ctrl

     ryan@quad4:~$ sudo flint -d 0000:07:00.0 query full
     Image type:            FS3
     FW Version:            12.25.1020
     FW Release Date:       30.4.2019
     Part Number:           06W1HY_0JJN39_Ax
     Description:           Mellanox ConnectX-4 Single Port EDR PCIE Adapter LP
     Product Version:       12.25.1020
     Rom Info:              type=PXE version=3.5.701 cpu=AMD64
     Description:           UID                GuidsNumber
     Base GUID:             ec0d9a030092a51a        4
     Base MAC:              ec0d9a92a51a            4
     Image VSD:             N/A
     Device VSD:            N/A
     PSID:                  DEL2180110032
     Security Attributes:   N/A
     Default Update Method: fw_ctrl
     ```

    2. quad03
      ```
      ryan@quad3:~/firmware/mellanox/cx4$ sudo flint -d 0000:06:00.0 query full
      Image type:            FS3
      FW Version:            12.21.3012
      FW Release Date:       25.2.2018
      MIC Version:           2.0.0
      Product Version:       rel-12_21_3012
      Rom Info:              type=PXE version=3.5.305 cpu=AMD64
      Description:           UID                GuidsNumber
      Base GUID:             506b4b0300131962        4
      Base MAC:              506b4b131962            4
      Image VSD:             N/A
      Device VSD:            N/A
      PSID:                  DEL2180110032
      Security Attributes:   N/A
      Default Update Method: Legacy
      ryan@quad3:~/firmware/mellanox/cx4$ sudo flint -d 0000:07:00.0 query full
      Image type:            FS3
      FW Version:            12.21.3012
      FW Release Date:       25.2.2018
      MIC Version:           2.0.0
      Product Version:       rel-12_21_3012
      Rom Info:              type=PXE version=3.5.305 cpu=AMD64
      Description:           UID                GuidsNumber
      Base GUID:             506b4b0300085b56        4
      Base MAC:              506b4b085b56            4
      Image VSD:             N/A
      Device VSD:            N/A
      PSID:                  DEL2180110032
      Security Attributes:   N/A
      Default Update Method: Legacy
      ryan@quad3:~/firmware/mellanox/cx4$ lspci | grep ConnectX-4
      06:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
      07:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
      ```

    3. quad2 (already at the correct model)
    

      ```bash
      ryan@quad2:~$ sudo flint -d 0000:06:00.0 query full
      Image type:            FS3
      FW Version:            12.28.2302
      FW Release Date:       29.8.2024
      Part Number:           MCX455A-ECA_Ax
      Description:           ConnectX-4 VPI adapter card; EDR IB (100Gb/s) and 100GbE; single-port QSFP28; PCIe3.0 x16; ROHS R6
      Product Version:       12.28.2302
      Rom Info:              type=UEFI version=14.21.17 cpu=AMD64,AARCH64
                            type=PXE version=3.6.102 cpu=AMD64
      Description:           UID                GuidsNumber
      Base GUID:             98039b0300a8f7de        4
      Base MAC:              98039ba8f7de            4
      Image VSD:             N/A
      Device VSD:            N/A
      PSID:                  MT_2180110032
      Security Attributes:   N/A
      Default Update Method: fw_ctrl
      ryan@quad2:~$ sudo flint -d 0000:07:00.0 query full
      Image type:            FS3
      FW Version:            12.28.2302
      FW Release Date:       29.8.2024
      Part Number:           MCX455A-ECA_Ax
      Description:           ConnectX-4 VPI adapter card; EDR IB (100Gb/s) and 100GbE; single-port QSFP28; PCIe3.0 x16; ROHS R6
      Product Version:       12.28.2302
      Rom Info:              type=UEFI version=14.21.17 cpu=AMD64,AARCH64
                            type=PXE version=3.6.102 cpu=AMD64
      Description:           UID                GuidsNumber
      Base GUID:             98039b0300c35f3a        4
      Base MAC:              98039bc35f3a            4
      Image VSD:             N/A
      Device VSD:            N/A
      PSID:                  MT_2180110032
      Security Attributes:   N/A
      Default Update Method: fw_ctrl

      ryan@quad2:~$ lspci | grep ConnectX-4
      06:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
      07:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
      ```

    5. quad1 (already at the correct model)
      ```bash
      ryan@quad-a:~$  sudo flint -d 0000:06:00.0 query full
      Image type:            FS3
      FW Version:            12.28.2302
      FW Release Date:       29.8.2024
      Part Number:           MCX455A-ECA_Ax
      Description:           ConnectX-4 VPI adapter card; EDR IB (100Gb/s) and 100GbE; single-port QSFP28; PCIe3.0 x16; ROHS R6
      Product Version:       12.28.2302
      Rom Info:              type=UEFI version=14.21.17 cpu=AMD64,AARCH64
                            type=PXE version=3.6.102 cpu=AMD64
      Description:           UID                GuidsNumber
      Base GUID:             ec0d9a030092a4fa        4
      Base MAC:              ec0d9a92a4fa            4
      Image VSD:             N/A
      Device VSD:            N/A
      PSID:                  MT_2180110032
      Security Attributes:   N/A
      Default Update Method: fw_ctrl
      ryan@quad-a:~$  sudo flint -d 0000:07:00.0 query full
      Image type:            FS3
      FW Version:            12.28.2302
      FW Release Date:       29.8.2024
      Part Number:           MCX455A-ECA_Ax
      Description:           ConnectX-4 VPI adapter card; EDR IB (100Gb/s) and 100GbE; single-port QSFP28; PCIe3.0 x16; ROHS R6
      Product Version:       12.28.2302
      Rom Info:              type=UEFI version=14.21.17 cpu=AMD64,AARCH64
                            type=PXE version=3.6.102 cpu=AMD64
      Description:           UID                GuidsNumber
      Base GUID:             ec0d9a0300891ade        4
      Base MAC:              ec0d9a891ade            4
      Image VSD:             N/A
      Device VSD:            N/A
      PSID:                  MT_2180110032
      Security Attributes:   N/A
      Default Update Method: fw_ctrl
      ryan@quad-a:~$ lspci | grep ConnectX-4
      06:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
      07:00.0 Infiniband controller: Mellanox Technologies MT27700 Family [ConnectX-4]
      ```
      

2. Cross flash the adapter
```bash

sudo su
mst start

mst status

root@quad4:/home/ryan/firmware/mft-4.35.0-159-x86_64-deb# sudo flint -d /dev/mst/mt4115_pciconf0 -ocr hw set Flash0.WriteProtected=Disabled

-W- Firmware flash cache access is enabled. Running in this mode may cause the firmware to hang.
-I- Flash0 parameter was set successfully

root@quad4:/home/ryan/firmware/mft-4.35.0-159-x86_64-deb# 
flint -nofs --use_image_ps --ignore_dev_data -allow_psid_change -d /dev/mst/mt4115_pciconf0 -i /home/ryan/firmware/mellanox/cx4/fw-ConnectX4-rel-12_28_2302-MCX455A-FCA_Ax-UEFI-14.21.17-FlexBoot-3.6.102.bin burn
Done.
    Current FW version on flash:  N/A
    New FW version:               12.28.2302

Burn process will not be failsafe. No checks will be performed.
ALL flash, including the device data sections will be overwritten.
If this process fails, computer may remain in an inoperable state.

 Do you want to continue ? (y/n) [n] : y
Burning FW image without signatures - OK
Restoring signature                     - OK
-I- To load new FW, issue system-level reset.
```

2. Restart host
3. Reapply GUID/MAC as it was lost when cross flashed
> https://forums.servethehome.com/index.php?threads/lenovo-connectx-4-card-flash-to-mellanox.32434/#post-299653

   1. quad04
     ```bash
     flint -d 0000:06:00.0 -guid 506b4b0300085b7e -mac 506b4b085b7e -ocr sg
     flint -d 0000:07:00.0 -guid ec0d9a030092a51a -mac ec0d9a92a51a -ocr sg
     ```
   2. Quad03
    ```bash
    flint -d 0000:06:00.0 -guid 506b4b0300131962 -mac 506b4b131962 -ocr sg
    flint -d 0000:07:00.0 -guid 506b4b0300085b56 -mac 506b4b085b56 -ocr sg
    ```

### Bluefield

1. Check current firmware

```bash
mst start
flint -d /dev/mst/mt41686_pciconf0 q full
```

quad4
```bash
root@quad4:/home/ryan# flint -d /dev/mst/mt41686_pciconf0 q full
Image type:            FS4
FW Version:            24.42.1000
FW Release Date:       8.8.2024
Part Number:           MBF2M516A-CENO_Ax_Bx
Description:           BlueField-2 E-Series DPU 100GbE Dual-Port QSFP56; PCIe Gen4 x16; Crypto Disabled; 16GB on-board DDR; 1GbE OOB management; FHHL
Product Version:       24.42.1000
Rom Info:              type=UEFI Virtio net version=21.4.13 cpu=AMD64,AARCH64
                       type=UEFI Virtio blk version=22.4.13 cpu=AMD64,AARCH64
                       type=UEFI version=14.35.15 cpu=AMD64,AARCH64
                       type=PXE version=3.7.500 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             b8cef60300756ac2        6
Base MAC:              b8cef6756ac2            6
Image VSD:             N/A
Device VSD:            N/A
PSID:                  MT_0000000560
Security Attributes:   N/A
Default Update Method: fw_ctrl
Life cycle:            GA NON SECURED
Secure Boot Capable:   Enabled
```

quad3
```bash
root@quad3:/home/ryan# flint -d /dev/mst/mt41686_pciconf0 q full
Image type:            FS4
FW Version:            24.31.2006
FW Release Date:       31.8.2021
Part Number:           BF2M515A-VENOT_C01_Ax
Description:           BlueField-2 E-Series DPU 200GbE Single-Port QSFP56; PCIe Gen4 x16; Crypto Disabled; Secured Boot Disabled; 16GB on-board DDR; 1GbE OOB management; Tall Bracket; FHHL
Product Version:       24.31.2006
Rom Info:              type=UEFI Virtio net version=21.1.11 cpu=AMD64
                       type=UEFI Virtio blk version=22.1.11 cpu=AMD64
                       type=UEFI version=14.24.15 cpu=AMD64,AARCH64
                       type=PXE version=3.6.404 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             b8cef60300756918        6
Base MAC:              b8cef6756918            6
Image VSD:             N/A
Device VSD:            N/A
PSID:                  MT_0000000750
Security Attributes:   N/A
Default Update Method: fw_ctrl
Life cycle:            GA NON SECURED
Secure Boot Capable:   Enabled
```

quad2
```bash
root@quad2:/home/ryan/firmware/mellanox/bf2# flint -d /dev/mst/mt41686_pciconf0 q full
Image type:            FS4
FW Version:            24.31.2006
FW Release Date:       31.8.2021
Part Number:           BF2M515A-VENOT_C01_Ax
Description:           BlueField-2 E-Series DPU 200GbE Single-Port QSFP56; PCIe Gen4 x16; Crypto Disabled; Secured Boot Disabled; 16GB on-board DDR; 1GbE OOB management; Tall Bracket; FHHL
Product Version:       24.31.2006
Rom Info:              type=UEFI Virtio net version=21.1.11 cpu=AMD64
                       type=UEFI Virtio blk version=22.1.11 cpu=AMD64
                       type=UEFI version=14.24.15 cpu=AMD64,AARCH64
                       type=PXE version=3.6.404 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             b8cef6030075654c        6
Base MAC:              b8cef675654c            6
Image VSD:             N/A
Device VSD:            N/A
PSID:                  MT_0000000750
Security Attributes:   N/A
Default Update Method: fw_ctrl
Life cycle:            GA NON SECURED
Secure Boot Capable:   Enabled
```

quad1
```bash
root@quad-a:/home/ryan# flint -d /dev/mst/mt41686_pciconf0 q full
Image type:            FS4
FW Version:            24.31.2006
FW Release Date:       31.8.2021
Part Number:           BF2M515A-VENOT_C01_Ax
Description:           BlueField-2 E-Series DPU 200GbE Single-Port QSFP56; PCIe Gen4 x16; Crypto Disabled; Secured Boot Disabled; 16GB on-board DDR; 1GbE OOB management; Tall Bracket; FHHL
Product Version:       24.31.2006
Rom Info:              type=UEFI Virtio net version=21.1.11 cpu=AMD64
                       type=UEFI Virtio blk version=22.1.11 cpu=AMD64
                       type=UEFI version=14.24.15 cpu=AMD64,AARCH64
                       type=PXE version=3.6.404 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             b8cef60300756552        6
Base MAC:              b8cef6756552            6
Image VSD:             N/A
Device VSD:            N/A
PSID:                  MT_0000000750
Security Attributes:   N/A
Default Update Method: fw_ctrl
Life cycle:            GA NON SECURED
Secure Boot Capable:   Enabled
```

1. Get firmware

```bash
mkdir -p firmware/mellanox/bf2
pushd firmware/mellanox/bf2
  # fw-BlueField-2-rel-24_42_1000-MBF2M516A-CENO_Ax_Bx-NVME-20.4.1-UEFI-21.4.13-UEFI-22.4.13-UEFI-14.35.15-FlexBoot-3.7.500.bin.zip
  wget https://www.mellanox.com/downloads/firmware/fw-BlueField-2-rel-24_42_1000-MBF2M516A-CENO_Ax_Bx-NVME-20.4.1-UEFI-21.4.13-UEFI-22.4.13-UEFI-14.35.15-FlexBoot-3.7.500.bin.zip

  unzip fw-BlueField-2-rel-24_42_1000-MBF2M516A-CENO_Ax_Bx-NVME-20.4.1-UEFI-21.4.13-UEFI-22.4.13-UEFI-14.35.15-FlexBoot-3.7.500.bin.zip

  wget https://content.mellanox.com/BlueField/BFBs/Ubuntu24.04/bf-bundle-3.3.0-202_26.01_ubuntu-24.04_64k_prod.bfb
popd
```


```
PCI_ID=0000:01:00.0

#
# Grab the device info
#
root@quad3:/home/ryan/firmware/mellanox/bf2# flint -d ${PCI_ID} q full
Image type:            FS4
FW Version:            24.31.2006
FW Release Date:       31.8.2021
Part Number:           BF2M515A-VENOT_C01_Ax
Description:           BlueField-2 E-Series DPU 200GbE Single-Port QSFP56; PCIe Gen4 x16; Crypto Disabled; Secured Boot Disabled; 16GB on-board DDR; 1GbE OOB management; Tall Bracket; FHHL
Product Version:       24.31.2006
Rom Info:              type=UEFI Virtio net version=21.1.11 cpu=AMD64
                       type=UEFI Virtio blk version=22.1.11 cpu=AMD64
                       type=UEFI version=14.24.15 cpu=AMD64,AARCH64
                       type=PXE version=3.6.404 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             b8cef60300756918        6
Base MAC:              b8cef6756918            6
Image VSD:             N/A
Device VSD:            N/A
PSID:                  MT_0000000750
Security Attributes:   N/A
Default Update Method: fw_ctrl
Life cycle:            GA NON SECURED
Secure Boot Capable:   Enabled

#
# Flash
#

NEW_FIRMWARE_BIN=fw-BlueField-2-rel-24_42_1000-MBF2M516A-CENO_Ax_Bx-NVME-20.4.1-UEFI-21.4.13-UEFI-22.4.13-UEFI-14.35.15-FlexBoot-3.7.500.bin


#
# Double check the PSID is changing as expected.
#

sudo mstflint -d "${PCI_ID}" -i "${NEW_FIRMWARE_BIN}" burn

root@quad3:/home/ryan/firmware/mellanox/bf2# sudo mstflint -d "${PCI_ID}" -i "${NEW_FIRMWARE_BIN}" burn

    Current FW version on flash:  24.31.2006
    New FW version:               24.42.1000


-E- PSID mismatch. The PSID on flash (MT_0000000750) differs from the PSID in the given image (MT_0000000560).

# do it for real

sudo mstflint -d "${PCI_ID}" -i "${NEW_FIRMWARE_BIN}" -allow_psid_change burn

root@quad3:/home/ryan/firmware/mellanox/bf2# sudo mstflint -d "${PCI_ID}" -i "${NEW_FIRMWARE_BIN}" -allow_psid_change burn
Done.
    Current FW version on flash:  24.31.2006
    New FW version:               24.42.1000


    You are about to replace current PSID on flash - "MT_0000000750" with a different PSID - "MT_0000000560".
    Note: It is highly recommended not to change the PSID.

 Do you want to continue ? (y/n) [n] : y
Burning FW image without signatures - OK
Burning FW image without signatures - OK
Restoring signature                     - OK
-I- To load new FW run mstfwreset or reboot machine.


# Restart

root@quad3:/home/ryan# flint -d 0000:01:00.0 q full
Image type:            FS4
FW Version:            24.42.1000
FW Release Date:       8.8.2024
Part Number:           MBF2M516A-CENO_Ax_Bx
Description:           BlueField-2 E-Series DPU 100GbE Dual-Port QSFP56; PCIe Gen4 x16; Crypto Disabled; 16GB on-board DDR; 1GbE OOB management; FHHL
Product Version:       24.42.1000
Rom Info:              type=UEFI Virtio net version=21.4.13 cpu=AMD64,AARCH64
                       type=UEFI Virtio blk version=22.4.13 cpu=AMD64,AARCH64
                       type=UEFI version=14.35.15 cpu=AMD64,AARCH64
                       type=PXE version=3.7.500 cpu=AMD64
Description:           UID                GuidsNumber
Base GUID:             b8cef60300756918        6
Base MAC:              b8cef6756918            6
Image VSD:             N/A
Device VSD:            N/A
PSID:                  MT_0000000560
Security Attributes:   N/A
Default Update Method: fw_ctrl
Life cycle:            GA NON SECURED
Secure Boot Capable:   Enabled


# Now flash the OS
systemctl enable rshim
systemctl start rshim

sudo apt install rshim pv

bfb-install --bfb bf-bundle-3.3.0-202_26.01_ubuntu-24.04_64k_prod.bfb --rshim rshim0
```

## IB Testing

### Ping pong
> https://www.hpc.temple.edu/mhpc/hpc-technology/exercise9/ibtests.html#ib-rdma-ping-pong


Bandwidth

```bash
kubectl -n node-debug debug node/quad4 -it --profile=sysadmin --image=registry.local/scratch/node-debug:0.1.2

# Use ibstatus to figure interface names
ibstatus

ibv_rc_pingpong -d mlx5_1 -g 0 -i 1

root@quad4:/# ib_write_bw -F --report_gbits -d mlx5_3 -a

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_3
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: ON		Lock-free      : OFF
 ibv_wr* API     : ON		Using Enhanced Reorder      : OFF
 CQ Moderation   : 100
 CQE Poll Batch  : Dynamic
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x08 QPN 0x0107 PSN 0x2957cd RKey 0x1fcfbe VAddr 0x0077eb60ae0000
 remote address: LID 0x09 QPN 0x0107 PSN 0xfb1158 RKey 0x1fcfbe VAddr 0x00797cb6ba8000
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 8388608    5000             48.50              48.43  		     0.000722
---------------------------------------------------------------------------------------
```

```bash
root@quad3:/# ib_write_bw -F --report_gbits 10.128.0.204 -d mlx5_2 -a
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_2
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: ON		Lock-free      : OFF
 ibv_wr* API     : ON		Using Enhanced Reorder      : OFF
 TX depth        : 128
 CQ Moderation   : 100
 CQE Poll Batch  : Dynamic
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x09 QPN 0x0107 PSN 0xfb1158 RKey 0x1fcfbe VAddr 0x00797cb6ba8000
 remote address: LID 0x08 QPN 0x0107 PSN 0x2957cd RKey 0x1fcfbe VAddr 0x0077eb60ae0000
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 2          5000           0.025165            0.022139            1.383670
 4          5000             0.33               0.25   		     7.784247
 8          5000             0.66               0.51   		     8.023243
 16         5000             1.33               1.10   		     8.563590
 32         5000             2.64               2.00   		     7.826571
 64         5000             5.09               4.05   		     7.909589
 128        5000             10.28              8.16   		     7.968694
 256        5000             19.21              16.99  		     8.297379
 512        5000             32.49              26.17  		     6.389036
 1024       5000             43.44              38.59  		     4.710744
 2048       5000             48.80              46.57  		     2.842351
 4096       5000             49.64              47.77  		     1.457707
 8192       5000             49.61              48.33  		     0.737528
 16384      5000             49.82              49.35  		     0.376490
 32768      5000             49.64              49.40  		     0.188463
 65536      5000             49.83              49.72  		     0.094829
 131072     5000             49.97              49.88  		     0.047569
 262144     5000             49.96              49.92  		     0.023801
 524288     5000             48.46              48.38  		     0.011535
 1048576    5000             48.66              48.63  		     0.005798
 2097152    5000             48.70              48.37  		     0.002883
 4194304    5000             48.51              48.42  		     0.001443
 8388608    5000             48.50              48.43  		     0.000722
---------------------------------------------------------------------------------------
root@quad3:/#
```

Latency
```
root@quad4:/# ib_write_lat -F --report_gbits -d mlx5_3
Disabling dynamic polling

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write Latency Test
 Dual-port       : OFF		Device         : mlx5_3
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: OFF		Lock-free      : OFF
 ibv_wr* API     : ON		Using Enhanced Reorder      : OFF
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 220[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x08 QPN 0x0108 PSN 0xcbb4cf RKey 0x1fcebc VAddr 0x0059d6875a5000
 remote address: LID 0x09 QPN 0x0108 PSN 0x1775d9 RKey 0x1fcebc VAddr 0x0063b084d93000
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
 2       1000          1.18           10.15        1.22     	       1.41        	0.74   		7.55    		10.15
---------------------------------------------------------------------------------------
root@quad4:/#
```

root@quad3:/# ib_write_lat -F --report_gbits -d mlx5_2 10.128.0.204
Disabling dynamic polling
---------------------------------------------------------------------------------------
                    RDMA_Write Latency Test
 Dual-port       : OFF		Device         : mlx5_2
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: OFF		Lock-free      : OFF
 ibv_wr* API     : ON		Using Enhanced Reorder      : OFF
 TX depth        : 1
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 220[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x09 QPN 0x0108 PSN 0x1775d9 RKey 0x1fcebc VAddr 0x0063b084d93000
 remote address: LID 0x08 QPN 0x0108 PSN 0xcbb4cf RKey 0x1fcebc VAddr 0x0059d6875a5000
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
 2       1000          1.19           8.47         1.22     	       1.40        	0.72   		7.36    	8.47
---------------------------------------------------------------------------------------
root@quad3:/#



## Node problem detector

- Check if right number of nics exist
  - Also IB nics are named a certain way
  - Add a label/condition saying fabric is available. Don't taint as I like to power off switch 
- Make sure peer mem exists
- PLX switch downstrema ACS settings

## GPU Direct RDMA 

From ChatGPT troubleshooting session

```bash
BIOS:
  Above 4G Decoding: enabled
  Peer Memory Write Enable: enabled
  BME DMA Mitigation: disabled
  VT-d/IOMMU: disabled, or intel_iommu=off

Linux after boot:
  clear ACS on PLX downstream ports:
    03:08.0
    03:09.0
    03:10.0
    03:11.0

NCCL:
  NCCL_IB_HCA=mlx5_2,mlx5_3
  NCCL_NET_GDR_LEVEL=PIX
  NCCL_DMABUF_ENABLE=0
```


Check ACS settings
```
for dev in 03:08.0 03:09.0 03:10.0 03:11.0; do
  echo "== $dev =="
  sudo lspci -vvv -s "$dev" | grep -iE 'Access Control|ACSCap|ACSCtl' -A3
done
```

Apply ACS
```
for dev in 03:08.0 03:09.0 03:10.0 03:11.0; do
  sudo setpci -s "$dev" f2a.w=0000
done
```