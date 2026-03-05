# meta-rcar-demo
Demo correction for R-Car

# Boot

## U-Boot

```
env default -a && env delete bootargs && load mmc 0:1 ${loadaddr} fitImage && bootm ${loadaddr}
```

# Tips

## タッチパネルが動作しない

最新リビジョンでinput deviceとしてvirtio-tablet-pciが使われるようになったが、その設定ファイルがAndroid側にある。
古い環境を継続している場合、Android側が未更新でタッチ操作が効かないことがある。
以下は、Androidのデバイスファイル設定を更新して再度ビルドしなおすための手順。

```
git -C work_v4hsbc_xen/android/device/epam/aosp-xenvm-trout/ fetch github
git -C work_v4hsbc_xen/android/device/epam/aosp-xenvm-trout/ pull github android-15-xenvm-trout-ih-main
rm ./work_v4hsbc_xen/android/out/target/product/xenvm_trout_arm64/boot.img
./build.sh xxxxx
```

なお、旧来のidcファイルを別途adb pushで入れ込む方法を取れば別途インストールしなおす必要は無く、タッチパネルは動作するようになる。

## kernel startingでstackしてしまう。

以前はYocto用のU-Bootの使用で発生した問題だが、Xen側のATFを最新環境に変更することで解消された。
そのため、最新のXenをビルドすれば発生しない問題のはず。
もし、継続して問題が起きる場合は、下記の手順でU-Bootの更新がおすすめ。
```
setenv flash_loader_mmc 'load mmc 0:1 ${loadaddr} flash.bin && sf probe && sf update ${loadaddr} 0 ${filesize} && reset'
saveenv
run flash_loader_mmc
```

## USB/PCIe booting

現時点で、Yocto側での対応、FWの配布が行われていないため、内容は未保証。
ローカルでのブートでは問題ないことを確認済み。

1. U-Bootの環境変数の準備
```
setenv flash_pcie_fw_to_qspi_from_xen_mmc 'load mmc 0:2 ${loadaddr} lib/firmware/rcar_gen4_pcie.bin && sf probe; sf update ${loadaddr} 0x300000 ${filesize}'
setenv renesas_rcar_gen4_load_firmware 'run set_pcie_firmware_info && sf probe; sf read ${renesas_rcar_gen4_load_firmware_addr} 0x300000 ${renesas_rcar_gen4_load_firmware_size}'
setenv set_pcie_firmware_info 'setenv renesas_rcar_gen4_load_firmware_addr 0x54000000 && setenv renesas_rcar_gen4_load_firmware_size 0x8000'

setenv flash_nvme_xen 'pci e && nvme scan && tftp ${loadaddr} full.img.gz && gzwrite nvme 0 ${loadaddr} ${filesize} 400000 0'
setenv flash_usb_xen 'pci e && usb start && tftp ${loadaddr} full.img.gz && gzwrite usb 0 ${loadaddr} ${filesize} 100000 0'

setenv xen_nvme 'pci e && nvme scan && env delete bootargs && load nvme 0:1 ${loadaddr} fitImage && bootm ${loadaddr}#default#boot_dev=nvme0n1'
setenv xen_usb 'pci e && usb start && env delete bootargs && load usb 0:1 ${loadaddr} fitImage && bootm ${loadaddr}#default#boot_dev=sda'
```

2. (一度だけ実行で大丈夫なはず)QSPI flashへのPCIe firmwareの書き込み

下記コマンドで書き込む場合は事前にXenを書き込んだSDを接続しておくこと。
上級者の方は任意の手段でQSPIにFWバイナリを書き込んで頂いて大丈夫です。

```
run flash_pcie_fw_to_qspi_from_xen_mmc
```

3. NVMe SSDの例) Xenのバイナリをtftp経由で書き込む。

```
run flash_nvme_xen
```

4. NVMe SSDの例) NVMeからXenをブートする

```
run xen_nvme
```

## Android(DomA)のユーザーイメージ領域の拡大

android/device/epam/aosp-xenvm-trout/xenvm_trout_arm64/BoardConfig.mkの
TARGET_USERDATAIMAGE_PARTITION_SIZEを任意の値に変更する

```
Ex.)
TARGET_USERDATAIMAGE_PARTITION_SIZE := 7516192768 # 7 GB
↓
TARGET_USERDATAIMAGE_PARTITION_SIZE := 19327352832 # 18 GB
```

boot.imgを削除してAndroidを再ビルドする。
```
rm work_v4hsbc_xen/android/out/target/product/xenvm_trout_arm64/boot.img
```


## DomD dtbへのdt-overlayの有効化

bootmに対するconfigを使うことでdtbにdtboを当てた状態でDomDを起動することができる。
dt_overlayに対して、カンマ「,」区切りにdtboファイル名を指定する必要あり。
ToDo: U-Boot側での自動判別機能、Yocto BSPと同等の入力機能の追加。

例: j1-imx219 + j2-imx708
```
bootm ${loadaddr}#default#dt_overlay=r8a779g3-sparrow-hawk-camera-j1-imx219.dtbo,r8a779g3-sparrow-hawk-camera-j2-imx708.dtbo
```

## Example: DomU + DomA Multi domain Multi display demo with SSD booting

1. Environment
   - Raspberry Pi Touch Display 2 7inch
   - M.2 SSD Boot
2. Build
```
./build.sh -v -u -a
```
3. Setup U-boot
```
setenv domd_conf '#dt_overlay=r8a779g3-sparrow-hawk-rpi-display-2-7in.dtbo'
setenv xen_nvme 'pci e && nvme scan && env delete bootargs && load nvme 0:1 ${loadaddr} fitImage && bootm ${loadaddr}#default#boot_dev=nvme0n1#doma_dev=/dev/nvme0n1p4${domd_conf}'
```
   - Points
      - domaのディスクはdoma_devを使ってU-Boot上であらかじめ指定する。
      - domdのdevicetreeへのdtboの有効化はdt_overlayにdtboファイル名を渡すことでDom0上で自動的に処理される。
4. Boot
```
run xen_nvme
```
## DomAのタッチデバイスに任意のディスプレイを使う方法

DSI:Waveshare 13.3インチタッチディスプレイ(解像度1920x1080)
DP-HDMI:12.3インチ横長タッチディスプレイ(解像度1920x720)
上記のディスプレイ構成で、DomAのディスプレイをWaveshare固定から横長のディスプレイに変更する方法。
これにより、それぞれのディスプレイでそれぞれのGuestDomainのタッチ操作が可能になる。

手順は以下の通り。
### 1. doma-virtio.cfgの編集
```
--- a/meta-xen-dom0/recipes-guests/doma/files/doma-virtio.cfg
+++ b/meta-xen-dom0/recipes-guests/doma/files/doma-virtio.cfg
@@ -35,6 +35,7 @@
 'backend=DomD, type=virtio,device, transport=pci, bdf=0000:00:05.0, grant_usage=0, backend_type=qemu',
 'backend=DomD, type=virtio,device, transport=pci, bdf=0000:00:06.0, grant_usage=0, backend_type=qemu',
 'backend=DomD, type=virtio,device, transport=pci, bdf=0000:00:07.0, grant_usage=0, backend_type=qemu',
+'backend=DomD, type=virtio,device, transport=pci, bdf=0000:00:08.0, grant_usage=0, backend_type=qemu',
 ]
 
 device_model_args=[
@@ -56,9 +57,10 @@
 '-device','virtconsole,chardev=virts_chardev6,id=virts6',
 '-device', 'virtio-net-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=4,romfile=,id=nic0,netdev=net0,mac=08:00:27:ff:cb:ce',
 '-netdev', 'type=tap,id=net0,ifname=vif-emu,br=xenbr0,script=no,downscript=no,vhost=on',
-'-device', 'virtio-net-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=6,romfile=,id=nic1,netdev=net1,mac=08:00:27:ff:cb:cf',
+'-device', 'virtio-net-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=8,romfile=,id=nic1,netdev=net1,mac=08:00:27:ff:cb:cf',
 '-netdev', 'type=tap,id=net1,ifname=vif-emu1,script=no,downscript=no,vhost=on',
-'-device', 'virtio-tablet-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=5',
+'-device', 'virtio-input-host-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=5,evdev=/dev/input/by-id/usb-wch.cn_TouchScreen_9LQ0172005164-event-if00',
+'-device', 'virtio-input-host-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=6,evdev=/dev/input/by-id/usb-wch.cn_TouchScreen_9LQ0172005164-if02-event-mouse',
 '-device', 'virtio-gpu-gl-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=7',
 '-display', 'sdl,gl=on',
 '-vga', 'std',
```
### 2. domu-virtio.cfgの編集
```
--- a/meta-xen-dom0/recipes-guests/domu/files/domu-virtio.cfg
+++ b/meta-xen-dom0/recipes-guests/domu/files/domu-virtio.cfg
@@ -43,7 +43,7 @@
 '-display', 'sdl,gl=on',
 '-vga', 'std',
 #'-device', 'vhost-vsock-pci,guest-cid=4,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=6',
-'-device', 'virtio-tablet-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=7',
+'-device', 'virtio-input-host-pci,disable-legacy=on,iommu_platform=on,bus=pcie.0,addr=7,evdev=/dev/input/touchscreen0',
 '-d', 'guest_errors',
 '-monitor', 'telnet:127.0.0.1:1235,server,nowait',
 '-global', 'virtio-mmio.force-legacy=false',
```
### 3. idcファイルのリネーム
```
(変更前)
work_v4hsbc_xen/android/device/epam/aosp-xenvm-trout/conf/Vendor_0627_Product_0003.idc
(変更後)
work_v4hsbc_xen/android/device/epam/aosp-xenvm-trout/conf/Vendor_27c0_Product_0859.idc
```
上記は12.3インチ横長タッチディスプレイを例にしたTips。
idcファイルの命名規則は以下のようになっており、VendroIDとProductIDを変えることで、任意のディスプレイをDomAのタッチディスプレイに設定可能。
```
Vendor_(VendorID)_Product_(ProductID).idc
```
### 4. work_v4hsbc_xen/android/device/epam/aosp-xenvm-trout/aosp_xenvm_trout_arm64.mkの編集
```
(変更前)
# Configure single touch device
PRODUCT_COPY_FILES += \
    device/epam/aosp-xenvm-trout/conf/Vendor_0627_Product_0003.idc:$(TARGET_COPY_OUT_VENDOR)/usr/idc/Vendor_0627_Product_0003.idc
(変更後)
# Configure single touch device
PRODUCT_COPY_FILES += \
    device/epam/aosp-xenvm-trout/conf/Vendor_27c0_Product_0859.idc:$(TARGET_COPY_OUT_VENDOR)/usr/idc/Vendor_27c0_Product_0859.idc
```

### 5. doma-set-rootの編集

DomUにDSI:Waveshare 13.3インチタッチディスプレイ(解像度1920x1080)、
DomAにDP-HDMI:12.3インチ横長タッチディスプレイ（解像度1920x720）
を割り当てるためのワークアラウンドとして、DomAの起動を遅延するパッチを当てる。
```
--- a/meta-xen-dom0/recipes-guests/doma/files/doma-set-root
+++ b/meta-xen-dom0/recipes-guests/doma/files/doma-set-root
@@ -50,3 +50,4 @@ if [ -n "$DOMA_STORAGE" ] ; then
     sed -i "s|/dev/${STORAGE_PART}3|${DOMA_STORAGE}|g" $DOMA_CFG_FILE
 fi
 
+sleep 30s
```
### 6. 再ビルド
```
./build.sh -v -u -a
```
