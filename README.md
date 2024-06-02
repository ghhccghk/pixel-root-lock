# 如何让pixel 设备（大概2代之后）实现bootloader处于锁定的情况下保证有root

### 注意：确认自己有能力不会手残，且会基础命令，了解adb，否则砖块很难救（pixel 6 以上 ）

## 1.首先准备好pixel设备的完整ota镜像，adb，[pixel 6 pro的](https://developers.google.cn/android/ota?hl=zh-cn#raven),avbroot 和 [magiskboot](https://github.com/ookiineko/magiskboot_build/tree/last-ci) ota镜像也可以更换成你想要用的系统的卡刷包。

## 2.准备编译magiskboot （linux需要编译一下）编译命令如下:

```
安装编译包（archlinux）
sudo pacman -S --needed base-devel xz lz4 bzip2 zlib pkgconf \
                        clang libc++ cmake ninja rust

编译命令

CC=clang CXX=clang++ cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release  # configure
cmake --build build -j $(nproc)  # build
./build/magiskboot  # running
# install to system (may need sudo, to specify different install dir, set the `DESTDIR' environment variable)
cmake --install build
```

## 2. 生成avb，ota，以及接下来刷入的avb秘钥，还有ota升级证书

### 建立文件夹 
```
mkdir ~/avbboot

cd ~avbboot

```

### 生成avb，ota秘钥 

```
avbroot key generate-key -o avb.key
avbroot key generate-key -o ota.key
```

### 将 AVB 签名密钥的公钥部分转换为 AVB 公钥元数据格式。这是引导加载程序在设置自定义信任根时所需的格式。

```
avbroot key extract-avb -k avb.key -o avb_pkmd.bin
```

### 生成ota升级证书

```
avbroot key generate-cert -k ota.key -o ota.crt
```

### 生成签名的ota包

```
avbroot ota patch \
    --input /path/to/ota.zip \
    --key-avb /path/to/avb.key \
    --key-ota /path/to/ota.key \
    --cert-ota /path/to/ota.crt \
	--prepatched /这段路径为想修补过的boot镜像，用来获取root的
```
###### ota.zip 为下载的ota包 ota.crt 为上面生成的ota证书 ota.key 为上面生成的ota私钥证书

####   出现这个就是表明生成成功了
```
20.870s  INFO Successfully patched OTA
```
#### 生成的文件为源文件名多.patch

### 验证签名的ota包

```
avbroot ota verify \
    --input /path/to/ota.zip \
    --cert-ota /path/to/ota.crt \
    --public-key-avb /path/to/avb_pkmd.bin
```

####   出现这个就是验证通过
```
20.870s  INFO Successfully patched OTA
```

###### avb_pkmd.bin 为引导加载程序在设置自定义信任根时所需的文件，如果省略--cert-ota 和--public-key-avb 选项，则只检查签名的有效性，而不检查签名是否可信。

### 解包刚才打patch的ota包

```
avbroot ota extract \
    --input /path/to/ota.zip.patched \
    --directory extracted \
    --fastboot
```

## 3.准备安装证书和镜像

### 将手机切换到bootloader模式

![就像这样](file:///tmp/Spectacle.LICPyN/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE_20240602_160740.png)

### 安装证书

```
fastboot erase avb_custom_key
fastboot flash avb_custom_key /path/to/avb_pkmd.bin
```

### 刷入镜像

```
ANDROID_PRODUCT_OUT=刚才解压patch的压缩包位置 fastboot flashall --skip-reboot``

```

## 4. 锁定bootloader之前，重新启动一次安卓系统，以确认所有内容都已正确签名。.

### 安装 Magisk 或 KernelSU 应用程序并运行以下命令：

```
adb shell su -c 'dmesg | grep libfs_avb'
```

### 如果 AVB 工作正常，则应打印出以下信息：

```
 init: [libfs_avb]Returning avb_handle with status: Success
```

## 5.锁定bootloader

```
fastboot flashing lock
```

# 恭喜这样就能成功了，然后看下成功启动视频，再见拜拜

