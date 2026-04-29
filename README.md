# Panther X2 (RK3566) Mainline Kernel Builder

用 **Linux 主线源码**（`torvalds/linux` stable 镜像）编译 Panther X2 内核。  
DTS 来源：从 ophub/amlogic-s9xxx-armbian 官方 DTB 反编译而来。

## 仓库结构

```
.
├── .github/workflows/
│   └── build-kernel.yml          # GitHub Actions 编译流程
├── dts/
│   └── rk3566-panther-x2.dts    # 板级设备树（从官方 DTB 反编译）
├── kernel-config/
│   └── panther-x2.fragment      # 在 rockchip_defconfig 上追加的配置项
└── README.md
```

## 快速开始

### 方法一：GitHub Actions（推荐，无需本地环境）

1. Fork 本仓库
2. 进入 **Actions** → **Build Mainline Kernel for Panther X2**
3. 点击 **Run workflow**，填写内核版本（如 `v6.9.4`）后运行
4. 编译完成后在 **Artifacts** 下载产物

### 方法二：本地编译

```bash
# 安装工具链
sudo apt-get install -y gcc-aarch64-linux-gnu make bc flex bison \
    libssl-dev libelf-dev device-tree-compiler

# 拉取内核
git clone --depth=1 --branch v6.9.4 \
    https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git

# 注入 DTS
cp dts/rk3566-panther-x2.dts linux/arch/arm64/boot/dts/rockchip/
echo 'dtb-$(CONFIG_ARCH_ROCKCHIP) += rk3566-panther-x2.dtb' \
    >> linux/arch/arm64/boot/dts/rockchip/Makefile

# 配置
cd linux
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- rockchip_defconfig
./scripts/kconfig/merge_config.sh -m .config ../kernel-config/panther-x2.fragment
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig

# 编译
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image modules dtbs
```

## 产物说明

| 文件 | 说明 | 设备路径 |
|------|------|---------|
| `boot-*.tar.gz` | 内核镜像 `Image` | `/boot/Image` |
| `dtb-*.tar.gz` | `rk3566-panther-x2.dtb` | `/boot/dtb/rockchip/` |
| `modules-*.tar.gz` | 内核模块 `.ko` | `/lib/modules/<version>/` |

## 安装到设备

```bash
# 在 Panther X2 上执行（已运行 Armbian/Linux）
cd /tmp

# 解压
tar xzf boot-*.tar.gz
tar xzf dtb-*.tar.gz
tar xzf modules-*.tar.gz

# 备份原内核
cp /boot/Image /boot/Image.bak

# 替换
cp boot/Image /boot/Image
cp dtb/rk3566-panther-x2.dtb /boot/dtb/rockchip/  # 路径按实际调整
cp -r modules/lib/modules/* /lib/modules/

# 更新 initramfs（如有）
update-initramfs -u

# 重启
reboot
```

## 支持的硬件

| 外设 | 状态 | 说明 |
|------|------|------|
| 以太网 (×2) | ✅ | RGMII, DWMAC |
| Wi-Fi | ✅ | BCM43430 SDIO，需固件 `brcmfmac43430-sdio.*` |
| 蓝牙 | ✅ | BCM43430A1 UART，需固件 `BCM43430A1.hcd` |
| USB 2.0 | ✅ | EHCI/OHCI |
| USB 3.0 (OTG) | ✅ | DWC3 host mode |
| eMMC | ✅ | DWCMSHC |
| TF 卡 | ✅ | DW-MSHC |
| LED (×4) | ✅ | pwr/wifi/eth/status |
| SPI | ✅ | 可用于 LoRa SX1302 |
| I2C | ✅ | PMIC、传感器 |
| UART | ✅ | 控制台 ttyS2@1500000 |
| GPU (Mali) | ⚠️ | 主线驱动功能有限，NPU 不支持 |
| HDMI | ⚠️ | VOP 驱动主线支持基础功能 |

### Wi-Fi / 蓝牙固件

主线内核不包含固件，需单独安装：

```bash
# Debian/Ubuntu
sudo apt-get install firmware-brcm80211

# 或从设备原系统复制
scp root@old-device:/lib/firmware/brcm/ /lib/firmware/brcm/
```

## 版本建议

| 内核版本 | 稳定性 | 说明 |
|---------|--------|------|
| `v6.9.4` | ⭐⭐⭐ | 社区验证最稳定，以太网/WiFi 均正常 |
| `v6.6.y` | ⭐⭐⭐ | LTS，长期支持 |
| `v6.12.y` | ⭐⭐ | 较新，驱动更完善但测试较少 |
