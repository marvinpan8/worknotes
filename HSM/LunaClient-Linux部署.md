# HSM-Lunaclient-Linux部署

- ubantu:  24.04
- LunaClient:  10.9.1
- 硬件图片 https://www.thalesdocs.com/gphsm/luna/7/docs/network/Content/install/network_hw_install/received_items_sa.htm

## 安装 Lunaclient

- 在 192.168.1.24/23 机器上
- 610-000397-015_SW_Linux_Luna_Client_V10.9.0_RevB.tar
- 先安装 ubantu 转换软件：

```properties
# 先安装 ubantu 转换软件
sudo apt-get install build-essential alien
# 解压
tar -xvf 610-000397-015_SW_Linux_Luna_Client_V10.9.0_RevB.tar
cd LunaClient_10.9.0-65_Linux/64
sudo sh install.sh help
# install.sh -p [network|pci|usb|backup|ped] [-c sdk|jsp|jcprov|snmp|fmsdk|fm_tools]
# -install_directory 安装目录，默认 /usr/safenet/lunaclient
# 1. pci卡安装
sudo sh install.sh -p pci
# 2. network卡安装
sudo sh install.sh -p network
# 将当前用户 ekemp 添加到 hsmusers 用户组
sudo gpasswd --add ekemp hsmusers
# 必须cp，sudo lunacm 才能正常运行，vtl 工具创建和交换证书
ll /usr/local/bin/
sudo cp /usr/safenet/lunaclient/bin/lunacm /usr/local/bin/
sudo cp /usr/safenet/lunaclient/bin/vtl /usr/local/bin/

# 由于所有容器都具有相同的 IP 地址，并且显示为同一客户端，因此必须在Luna Network HSM 7设备上禁用 ntls ipchecking
# ntls ipcheck disable
```

## 添加用户组

```properties
# 查看1002用户组为空
cat /etc/group |grep 1002
# 添加用户组 hsmusers， GID=1002
groupadd -g 1002 hsmusers
# 若已存在，修改组的 GID=1002
groupmod -g 1002 hsmusers
# 查看组信息，显示 1002 就是 GID
getent group hsmusers
# 或查看 /etc/group
grep hsmusers /etc/group
# 添加用户到 组 
adduser ekemp hsmusers
```

## 拷贝证书和配置SO

```properties
# 检查配置文件
sudo ls -l /etc/Chrystoki.conf.debsave
# 拷贝配置文件
sudo cp -r cert /usr/safenet/lunaclient/
sudo cp cli082-yken/Chrystoki.conf /etc/
# 将 libLunaAPI 放置在 jdk/lib 目录下 
ll /usr/safenet/lunaclient/jsp/lib
cp /usr/safenet/lunaclient/jsp/lib/libLunaAPI.so /usr/local/jdk-11.0.30/lib/
```

## 测试验证

二进制文件工具介绍：https://www.thalesdocs.com/gphsm/luna/7/docs/network/Content/install/client_install/linux_minimal_install_overview-and-prep.htm

- **lunacm**：分区管理工具
- **vtl**：配置工具（证书创建和交换、客户端分区注册、日志记录等）
- **configurator**：配置文件管理工具
- **ckdemo**：演示 HSM 中的单个原子 PKCS#11 操作
- **cmu**：证书管理实用程序
- **mkfm**：允许客户端连接到功能模块（如果您已在 HSM 中安装任何功能模块）
- **pscp/plink**：用于一步式 NTLS
- **salogin**：持久应用程序连接工具
- **multitoken**：在多个插槽上执行多个加密命令

```properties
# 查看安装信息
sudo vtl supportInfo
vim c_supportInfo.txt
# 登录查看Available HSMs
sudo lunacm
# 登录输入秘钥 ykenXSW@co
role login -n co
# 查看key
par con
# Policy 41 0=分区V0 1=分区V1
par showPolicies
# 查看 CPv4；策略号42: Allow CPv1=0；Luna Cloud HSM 策略号为8 
par ciphershow
# 修改允许 CPV4
par changePolicy -policy 42 -value 0
# 44: Allow Extended Domain Management : 0  允许扩展域；
# 列出当前分区所有的克隆域
par domainlist
exit
```

## java11安装

```properties
tar -zxvf jdk-11.0.30_linux-x64_bin.tar.gz
sudo mv jdk-11.0.30 /usr/local/
sudo vim /etc/profile
#-----最后行添加-------------------------------
export JAVA_HOME=/usr/local/jdk-11.0.30
export CLASSPATH=$:CLASSPATH:$JAVA_HOME/lib/
export PATH=$PATH:$JAVA_HOME/bin
#--------------------------------------------
source /etc/profile
java -version
#------中间添加 LunaProvider -------------------------------
vim $JAVA_HOME/conf/security/java.security
#--------------------------------------------
security.provider.13=com.safenetinc.luna.provider.LunaProvider
#--------------------------------------------
#拷贝 hsm-test-1.0-SNAPSHOT-jar-with-dependencies.jar 到 ~/hsm 测试
cd ~/hsm
java -cp /usr/safenet/lunaclient/jsp/lib/LunaProvider.jar:hsm-test-1.0-SNAPSHOT-jar-with-dependencies.jar com.ekemp.hsm.sample.LunaPkcs11AttributesDemo
```



# 卸载 lunaclient

> **注意：移除单个JSP 组件或 SDK 组件，必须先完全卸载Luna HSM 客户端，然后重新运行安装脚本，不要选择不需要的组件。**

```properties
cd /usr/safenet/lunaclient/bin
sudo sh uninstall.sh
```

## 内核升级的影响

如果在成功安装Luna HSM 客户端后升级 Linux 内核，则必须安装新内核的内核头文件，并重新编译 UHD、K6 和 K7 驱动程序。新内核将在重启后生效。

##### 更新内核，然后使系统恢复到就绪状态：

1. 如果尚未安装，请安装开发工具。
2. 如有需要，请更新内核。再重启。
3. 安装新内核的内核头文件，例如：**yum install kernel-headers-$(uname -r)**
4. 为新内核重新构建驱动程序：**rpmbuild --rebuild uhd-7.3.0-165.src**
5. 对 k6 和 k7 驱动程序执行相同的操作。