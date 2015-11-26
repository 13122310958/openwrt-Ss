# 打造一台翻墙路由器

### 凡是连上wifi的手机电脑，所有终端无需任何设置，都能自动翻墙。

路由器配置方案：ShadowSocks-libev + ChinaDNS

本教程以Dlink dir-505为例，其他型号路由也类似，Openwrt版本：14.07。

首先路由器型号需要在openwrt列表中：http://wiki.openwrt.org/toh/start/
(可以ctrl+F搜索匹配型号)，并记录所用路由器cpu的型号。

### 一.下载路由器CPU型号对应的的固件：https://downloads.openwrt.org/

### 二.刷机及配置
  * 访问路由器上传固件刷机（一般在路由器系统设置里面），等待一会儿，勿断电。
  * 好了之后打开wifi开关，连上openwrt（有的固件默认不没有开启WI-FI需要用网线连接），访问192.168.1.1
  * System－》Administertion，设置路由密码，选中Allow remote hosts to connect to local SSH forwarded ports
  * Network－》Wifi，WI-FI没有启动先启动，设置Transmit Power为20dBm（100mW），设置Country Code为CN-China
  * 设置下WIFI名称（ESSID）、密码、加密方式（Encryption,推荐使用WPA-PSK/WPA2-PSK）
  * Network－》Interfaces，修改LAN口，勾掉Bridge interfaces选项，更改网关为192.168.5.1，添加WAN口选择DHCP client选项（如果需要拨号选择PPPoE）,勾上Adapter “eth1”，设置Firewall Settings为WAN。注意：此处修改LAN口、添加WAN口不要“保存&应用”，先“保存”，在Network－》Interfaces列表整体“保存&应用”

**注意**: 为了和主路由不冲突最好将网关改为其他，比如192.168.5.1
### 刷机成功。

### 三.下载安装包并安装到路由器
* 下载安装包
  * Shadowsocks-spec - [http://sourceforge.net/projects/openwrt-dist/files/shadowsocks-libev/](http://sourceforge.net/projects/openwrt-dist/files/shadowsocks-libev/)
  * chinaDNS - [http://sourceforge.net/projects/openwrt-dist/files/chinadns/](http://sourceforge.net/projects/openwrt-dist/files/chinadns/)
  * Shadowsocks-spec-LuCI - [http://sourceforge.net/projects/openwrt-dist/files/luci-app/shadowsocks-spec/](http://sourceforge.net/projects/openwrt-dist/files/luci-app/shadowsocks-spec/)
  * chinaDNS-LuCI - [http://sourceforge.net/projects/openwrt-dist/files/luci-app/chinadns/](http://sourceforge.net/projects/openwrt-dist/files/luci-app/chinadns/)
 
  **注意**: 一定要下载路由器cpu型号对应的文件！dir-505选择ar71xx
* 安装到路由器
  * 保证路由器可以上外网
  * 用ssh工具上传shadowsocks-libev-spec、ChinaDNS、及2个luci文件 4个ipk包到路由器/tmp目录
  ```
  ssh root@192.168.5.1
  ```
  ```
  opkg update
  ```
* 安装中文包，然后在路由管理界面System－》System－》Language and Style切换中文语音，刷新可看到中文
    ```
    opkg install luci-i18n-chinese
    ```
    ```
    cd /tmp
    ```
    ```
    opkg install shadowsocks-libev-spec-polarssl_2.2.2-1_ar71xx.ipk
    ```
    ```
    opkg install ChinaDNS_1.3.1-1_ar71xx.ipk
    ```
    ```
    opkg install luci-app-shadowsocks-spec_1.3.2-1_all.ipk
    ```
    ```
    opkg install luci-app-chinadns_1.3.1-1_all.ipk
    ```

  **注意**: 期间可能会遇到错误提示,没关系，这时因为安装ipset包后需要重启。

* 编辑配置文件，输入shadowsocks服务器信息
    ```
    vi /etc/shadowsocks/config.json
    ```
    ```
    {
        "server":"你的服务器Ip",
        "server_port":端口,
        "local_port":1080,
        "password":"密码",
        "timeout":300,
        "method":"aes-256-cfb"
    }
    ```
    也可在服务－》ShadowSocks 全局设置中输入S’s账号（先去掉使用配置文件，S’s账号需要和服务配置匹配）

* 更改配置，进入路由器管理界面(http://192.168.5.1)
  * 网络－> DHCP/DNS，基本设置里将本地服务器更改为：127.0.0.1#5353，HOSTS和解析文件里勾选“忽略解析文件”和“忽略HOSTS文件”
  * 服务－> ChinaDNS里ChinaDNS上游服务器更改为：114.114.114.114,127.0.0.1:5300 (适合服务器支持UDP转发，较新版本Shadowsocks均支持)

### 四.设置自动更新忽略配置文件。
敌人的策略一直在变化，为了实现智能分流需要设置自动更新忽略配置文件，长时间不更新可能导致部分国内网站走代理或者国外不走代理，如下：

* 新建自动更新配置文件，内容如下：
  ```
  vi /root/update.sh
  ```
  ```
  #!/bin/sh
  
  ping -c2 www.baidu.com >>/dev/null
  if [ $? -eq 0 ];then
  	wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/shadowsocks/ignore.list
  	wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chinadns_chnroute.txt
  else
          echo 'network is down...'
  fi
  ```
* 然后用命令给予该脚本执行权限并执行，正常的话配置文件会成功更新
  ```
  chmod +x /root/update.sh
  ```
  ```
  /root/update.sh
  ```

* 在路由器管理界面点击“系统”→“计划任务”中，输入以下内容：
  ```
  30 4 * * * /root/update.sh
  ```
保存后，每天的凌晨4点半就会自动执行更新脚本了。

* 重启路由器，系统－》重启

### 翻墙成功，体验高速穿越吧！

   **如果觉得以上方式自己搞不定或有任何问题欢迎进q群讨论：194309856，也提供各种做好的路由器插上电源直接翻墙，淘宝购买连接：https://shop141824031.taobao.com/index.htm**
