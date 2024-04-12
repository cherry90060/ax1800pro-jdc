**京东云亚瑟路由器TTL刷u-boot升级iStoreOS固件** 


1.拆机
撕开路由器底部胶垫，露出底部的5颗螺丝。用螺丝刀全部拧开。
路由器顶盖使用卡扣固定。敲开后可以看到顶部还有两个固定螺丝，拧开。注：网上很多拆机教程里没有提到这两颗螺丝，导致折腾很久都没能弄开。
用力挤压路由器筒壁，使网口缩进外壳。抽出路由器本体完成拆机。
2.搞定TTL接口
可以看到主板上4个并排的孔V R T G就是TTL接口。
去淘宝搜 TTL 探针，用的时候直接插到孔上。
焊接的针脚会抵住外壳导致无法塞回去，到时候还要去掉。
到网上买个 CH340 的 USB 转串口的转接卡。
我的转接卡上有 3v 和 5v 的跳线，短接 3v 。
转接卡和路由器接线时注意：G-G，R-T，T-R 这样接。V不用接。
G：接地
R：接收
T：发送
windows系统没有自带 CH340 的驱动，要去网上找个驱动安上。
刷入 u-boot
串口转接卡一头连好路由器，另一头插入电脑。
用网线连接路由器和电脑，并将电脑的 IP 设置为 192.168.10.1 。
在电脑的设备管理器中查看转接卡对应的串口名。
用 putty 连接串口，波特用 115200 。
路由器开机，按回车进入 6018# 模式。此时已经可以使用 boot loader 的命令了。
将 u-boot.mbn 复制到 Tftpd64 同一目录。启动 Tftpd64 。
注：此时 server interface 里的 IP 地址应为 192.168.10.1 。
u-boot 可以从 https://github.com/cherry90060/ax1800pro-jdc/blob/main/u-boot.mbn 这里下载。
中 putty 里敲入如下命令完成刷机
tftpboot u-boot.mbn && flash 0:APPSBL && flash 0:APPSBL_1
看到命令输出里有几个 `OK` 就完成 u-boot 刷机了。
3.准备 iStoreOS 固件
iStoreOS 是基于 OpenWRT 的一个路由器固件。固件可以从 https://fw.koolcenter.com/Lean/JDC_AX1800_Pro/ 下载到。我们需要下载其中的 *-kernel-rootfs.rar 固件。解压后会有两个 bin 文件。需要使用使用工具将两个文件合并成一个 bin 文件后一起刷入。合并工具可以使用 UBin 。由于没有找到软件的官网，大家还说自行搜索下载吧。

刷入 iStoreOS 固件
将电脑 IP 设置为 192.168.1.2 。
重启路由器并用 putty 重新连接路由器进入 6018# 模式。
输入命令 httpd 192.168.1.1 启动刷机的 web 服务。
看网上的教程，我以为 u-boot 启动的时候会自动启动刷机的 web 服务。在 u-boot 的 help 里看到 httpd 才知道刷机服务要手动启动。
使用浏览器打开网页 http://192.168.1.1 进入刷机的 web 界面。
上传固件开始刷机。刷机成功后会路由器会自动重启。路由器启动后路由器灯为绿色。
用浏览器重新打开 http://192.168.1.1 ，此时已经可以看到 iStoreOS 的登录界面了。
默认用户名为 root ，密码 password 。
4.安装Docker
我使用的 iStoreOS 固件并未集成 Docker 。尝试使用 is-opkg install docker dockerd docker-compose 安装 Docer 套件，提示空间不足，无法安装。查看磁盘信息， Overlay 分区只有几M，几乎无法安装任何东西。查看分区信息， mmcblk0p25 分区有 300M 空间。将 mmcblk0p25 分区里的数据删除，并将 /overlay 数据复制过来。在 iStoreOS 的 挂载点 中将 mmcblk0p25 挂载到 /overlay 。应用并重启路由器后可以看到 /overlay 的空间已经变成 300M 里。

安装好 Docker 后，/overlay 还能剩下大概 27M 的空间，几乎没有空间安装其他软件。考虑到其他软件都会使用 Docker 安装，因此暂不打算对 /overlay 做扩容了。


**亚瑟AX1800PRO 扩容overlay方案记录**

思路 mmcblk0p27太大了，保留200g，剩下的拆分出来，想怎么安排就自己安排。

lsblk 命令查看当前固件分区格式
cfdisk  /dev/mmcblk0   分区  网上有教程，把27删掉，划分为27 28，28我们自用。
需要重启路由器才生效

格式化之前要取消挂载 否则无法格式化
mkfs.ext4 /dev/mmcblk0p27
mkfs.ext4 /dev/mmcblk0p28
大盘格式化很慢，感觉这个方案应该不影响其他的。


挂载
mount /dev/mmcblk0p27 /mnt/mmcblk0p27
先创建 mmcblk0p28目录然后挂载
mount /dev/mmcblk0p28 /mnt/mmcblk0p28
拷贝overlay到28目录
cp -r /overlay/* /mnt/mmcblk0p28

ls /mnt/mmcblk0p28    检查是否拷贝成功，看到 lost+found  upper       work 说明拷贝成功。

打开WEB管理界面，点“挂载点”划到最下面找到添加
先取消overlay挂载，然后再挂载
        第一步：启用挂载点
        第二步：在uuid直接选中自己刚刚新建的分区
        第三步：挂载点选项卡，选“作为外部 overlay 使用”
        第四步： 保存并使用 （需要取消原来的 overlay 挂载点，原可以挂载loop0）
最后，reboot 重启OpenWrt。

删除WiFi（再软件包里面找到kmod-cfg80211-linux 移除，再用ssh登陆路由器 进入etc/config/wireless 删除即可，高版本wifi驱动可能是ath11相关模块）
