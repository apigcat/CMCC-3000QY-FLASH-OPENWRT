# CMCC-3000QY-FLASH-OPENWRT
便宜买的刷openwrt

直接用Bp抓日志http://192.168.8.1/itms

重发"fname":"websys.log|/usr/sbin/dropbear -p 22"#开启22端口

重发"fname":"websys.log|passwd root -d"#删除密码（没必要反弹shell了）

ssh -o HostKeyAlgorithms=+ssh-rsa root@192.168.8.1 #免除ssh验证密码就是

vi /etc/config/dropbear #这里修改端口

/etc/init.d/dropbear enable #开机自启22

刷机文件用的这里的
https://github.com/sfxfs/rax3000qy-OpenWrt/tree/main#
uboot刷入
  mtd write /tmp/nwrt_rax3000qy_uboot.mbn /dev/mtd11
  
  mtd write /tmp/nwrt_rax3000qy_mibib.bin /dev/mtd1

  电脑进入控制面板内的网络接口设置网口为静态 ip `192.168.1.2`一定要插WAN口【刷BIN文件】配置IP地址
遇到几个问题解决了
第一个 MAC混乱/etc/config/network文件添加修改以下代码
config interface 'lan'
	option type 'bridge'
	option proto 'static'
	option netmask '255.255.255.0'
	option multicast_querier '0'
	option igmp_snooping '0'
	option ipaddr '192.168.8.1'
	option ip6assign '64'
	option _orig_ifname 'eth0 ath0 ath1'
	option _orig_bridge 'true'
	option delegate '0'
	option ifname 'ath0 ath1 eth0'
	option force_link '1'
	# 只添加这行MAC地址修复
	option macaddr '94:BE:09:53:16:2D'

第二个问题不定时重启失败
---------------------------------------原脚本----------------------------------------------
#!/bin/sh /etc/rc.common
START=50  # 启动顺序太早，可能与关键服务冲突

run_reboot()
{
    local enable
    config_get_bool enable $1 enable

    if [ $enable = 1 ]; then
        local minute
        local hour
        config_get week $1 week
        config_get minute $1 minute
        config_get hour $1 hour
        
        # 问题1：格式处理逻辑不完整
        if [ $minute = 0 ] ; then
            minute="00"  # 仅处理0，其他单数字没处理
        fi
        
        # 问题2：week=7转*后，后续echo又使用$week变量
        if [ $week = 7 ] ; then
            week="*"  # 变量被覆盖，但下面echo仍用$week
        fi
        
        # 问题3：每次都清理重启任务，可能导致其他脚本的任务也被清理
        sed -i '/reboot/d' /etc/crontabs/root >/dev/null 2>&1
        /etc/init.d/cron restart  # 频繁重启cron服务
        
        # 问题4：硬编码sleep 5可能不够，reboot可能失败
        echo "$minute $hour * * $week sleep 5 && touch /etc/banner && reboot" >> /etc/crontabs/root
        echo "Auto REBOOT has started."
    else
        # 问题5：无论是否启用，都会执行清理和输出启动信息
        sed -i '/reboot/d' /etc/crontabs/root >/dev/null 2>&1
        /etc/init.d/cron restart  # 即使禁用也重启cron
        echo "Auto REBOOT has started."  # 逻辑错误：禁用时应输出停止信息
    fi
}

start()
{
    config_load autoreboot
    config_foreach run_reboot login  # 缺少配置验证
}

stop()
{
    echo "Auto REBOOT has stoped."  # 问题6：stop函数没有实际清理crontab
}
---------------------------------------原脚本----------------------------------------------



修改后
---------------------------------------精简修改----------------------------------------------
#!/bin/sh /etc/rc.common
START=99  # 延迟启动，避免冲突

validate_time() {
    [ "$1" -ge 0 -a "$1" -le 59 ] 2>/dev/null && return 0
    return 1
}

configure_cron_job() {
    local enable minute hour week cron_week
    
    config_get_bool enable "$1" enable 0
    [ $enable -eq 0 ] && return
    
    config_get minute "$1" minute 30
    config_get hour "$1" hour 4
    config_get week "$1" week 7
    
    # 验证时间参数
    validate_time "$minute" || minute=30
    [ "$hour" -ge 0 -a "$hour" -le 23 ] 2>/dev/null || hour=4
    
    # 格式化分钟（crontab要求两位）
    [ ${#minute} -eq 1 ] && minute="0$minute"
    
    # 星期转换（7表示每天）
    [ "$week" = "7" ] && cron_week="*" || cron_week="$week"
    
    # 只清理本脚本添加的任务（通过唯一标识）
    sed -i '/#AUTOREBOOT#/d' /etc/crontabs/root
    
    # 添加新任务（带唯一标识）
    echo "$minute $hour * * $cron_week sleep 60 && reboot #AUTOREBOOT#" >> /etc/crontabs/root
    
    logger "Autoreboot scheduled: $minute:$hour (week:$week)"
}

start_service() {
    [ -f /etc/config/autoreboot ] || return 0
    
    config_load autoreboot
    config_foreach configure_cron_job login
    
    # 仅当crontab有变化时才重启服务
    crontab -l | grep -q "#AUTOREBOOT#" && /etc/init.d/cron restart
}

stop_service() {
    # 精确清理，避免影响其他任务
    sed -i '/#AUTOREBOOT#/d' /etc/crontabs/root
    /etc/init.d/cron restart
    logger "Autoreboot stopped"
}

reload_service() {
    stop_service
    start_service
}
---------------------------------------精简修改----------------------------------------------


