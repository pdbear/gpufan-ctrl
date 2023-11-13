# gpufan-ctrl
PVE fan control script

关于被动散热GPU风扇的脚本说明：

这里皮蛋熊针对群友的三个主板做了实验：
1. 华擎 Z370M PRO4  -> NCT6683D
2. MSI B460M MORTAR WIFI -> NCT6687
3. 华硕B660M-PLUS  D4 -> NCT6798D

这里不同主板的控制芯片不同，比如华擎的Z370M使用的是`nct6683d`芯片，也有使用`IT87`的，以下教程需要根据自己的不同，处理芯片驱动，同时确保内核开启了如下选项：
```bash
CONFIG_THERMAL=y
CONFIG_THERMAL_HWMON=y
```
这里以`MSI B460M MORTAR WIFI` + `PVE`为例，进行说明。
我发现PVE的内核驱动目录中存在NCT的驱动：
```bash
/lib/modules/5.15.102-1-pve/kernel/drivers/hwmon/nct6683.ko
```
可以强制插入内核中：
```bash
insmod /lib/modules/5.15.102-1-pve/kernel/drivers/hwmon/nct6683.ko force=1
```
但会遇到`pwm`通道不可写的问题：
```bash
$ echo 1 > /sys/class/hwmon/hwmon2/pwm1
hwmon2/pwm1: Permission denied
```
如果强制给写权限，也会出现IO错误导致无法写入。

**注意**： 这是因为`nct6683`的驱动并不兼容`nct6687`导致的。

我们需要重新编译一份`nct6687`的驱动，可以参考这个大佬的仓库 [NCT6687D](https://github.com/Fred78290/nct6687d) 
编译好了过后，我们会得到`nct6687.ko`，此时将其插入到内核中，可正常驱动内核：
```bash
# 先 cd 到 nct6687.ko 所在目录
insmod nct6687.ko
```
此时在`/sys/class/hwmon/hwmonX`中可以找到相关的控制文件
```bash
root@pve:/sys/class/hwmon/hwmon4# ls -al
total 0
drwxr-xr-x 3 root root    0 Nov  7 20:41 .
drwxr-xr-x 3 root root    0 Nov  7 20:41 ..
lrwxrwxrwx 1 root root    0 Nov  7 20:41 device -> ../../../nct6687.2592
-r--r--r-- 1 root root 4096 Nov  7 20:41 fan1_input
-r--r--r-- 1 root root 4096 Nov  7 20:41 fan1_label
-r--r--r-- 1 root root 4096 Nov  7 20:41 fan1_max
-r--r--r-- 1 root root 4096 Nov  7 20:41 fan1_min
***
-r--r--r-- 1 root root 4096 Nov  7 20:41 name
drwxr-xr-x 2 root root    0 Nov 11 20:23 power
-rw-r--r-- 1 root root 4096 Nov 11 20:23 pwm1
-rw-r--r-- 1 root root 4096 Nov 11 20:23 pwm2
-rw-r--r-- 1 root root 4096 Nov 11 20:23 pwm3
-rw-r--r-- 1 root root 4096 Nov 12 11:11 pwm4
***
```
如果你不知道你的`hwmon`是哪一个，可以挨个目录切换进去 ，然后`cat name`，看看是否是你的输出，比如我这里输出：
```bash
root@pve:/sys/class/hwmon# cat ./hwmon1/name
nvme
root@pve:/sys/class/hwmon# cat ./hwmon2/name
coretemp
root@pve:/sys/class/hwmon# cat ./hwmon3/name
iwlwifi_1
root@pve:/sys/class/hwmon# cat ./hwmon4/name
nct6687
```
可以看到这里是`hwmon4`是我们的`nct6687`的`sys file`目录。
**注意**：这里的`hwmon4`并不是固定的，是由`hwmon`驱动的加载顺序决定的。
比如我这里是`hwmon4`，如果设置了开机自动加载驱动，那有可能就变为`hwmon2`了。

此时，我们可以准备一个主板的4pin风扇，开始准备寻找通道了。
```bash
root@pve:/sys/class/hwmon/hwmon4# cat fan1_label
CPU Fan
root@pve:/sys/class/hwmon/hwmon4# cat fan2_label
Pump Fan
root@pve:/sys/class/hwmon/hwmon4# cat fan3_label
System Fan #1
root@pve:/sys/class/hwmon/hwmon4# cat fan4_label
System Fan #2
root@pve:/sys/class/hwmon/hwmon4# cat fan5_label
System Fan #3
root@pve:/sys/class/hwmon/hwmon4# cat fan6_label
System Fan #4
```
可以看到`fan1`是CPU风扇，`fan2`是水冷风扇，`fan3~fan8`是机箱风扇。
注意：并不是所有主板都把这些通道引出了，比如大部分的可能只引出了两个机箱风扇，不过这也够我们用了。

这时候我们可以去观察下主板的标注：
![[Pasted image 20231112113657.png]]
红框处从左往右分别标注了`CHA_FAN1`,`CHA_FAN2`,`CHA_FAN3`，不同主板标注不一样，有的标注是`sys_fan1`,`sys_fan2`,`sys_fan3`等。

这里我们准备一个4Pin风扇，这里我以我买的的`7530 pwm风扇`为例:
![[Pasted image 20231112125615.png]]
![[Pasted image 20231112125621.png]]
将接线端插入主板的`cha_fan`中方便的那个，这里以`cha_fan2` 为例，
根据上面我们查询的知识知道`fan4_label`输出为`System Fan #2`，即是对应的`cha_fan2`。

如果你和我一样，是四线的风扇，正常的线序应该是`GND,VCC,PWM,INPUT`，最后一个`INPUT`是采集风扇转速的，你可以通过下面命令查看风扇转速：
```bash
# 具体的hwmon号，需要根据具体情况判断
cd /sys/class/hwmon/hwmon4

cat fan1_input
cat fan2_input # 这里可以取值根据你的驱动决定，这里我的是 fan1 ~ fan8
```
有了上面的输出，我们可以判断在哪些通道上采集到了转速信号，便于确定我们的风扇接到了哪个通道上。为了进一步确定我们的通道选择是否正确，此时我们可以操作下面命令：
```bash
# 具体的hwmon号，需要根据具体情况判断
cd /sys/class/hwmon/hwmon4
# 判断风扇是否停转
echo 0 > pwm4  
# 判断风扇是否满速运转
echo 255 > pwm4
```
如果上面指令起到效果了（观察风扇转速），那证明我们的控制链路已经通了，我们已经明确知道`pwm4`和`fan4`对应到我们插入的主板`cha_fan2`了。
**注意**：不同的主板接线方式不同，比如`华擎Z370M PRO4`这个主板就接反了线，结果发现其`fan4、PWM3`是一组的，如果你没找到输出，建议从`pwm1 ~ pwm8`（不同控制芯片的通道数不同）挨个执行上面指令，判断是否起了效果。

现在假设你已经确定了你主板上的`cha_fan2`可以被`hwmon4`的`fan4、pwm4`所控制了，那么我们就可以固化一下我们的设置了。

我们先将内核驱动设置为开机自启动：
```bash
cp nct6687.ko /lib/modules/$(uname -r)/
depmod -a

# vim /etc/modules 添加下面模块
nct6687

# 更新 initramfs
update-initramfs -u -k all

# 重启电脑
reboot
```
此时我们重启一下电脑，重新检查一下`/sys/class/hwmon`目录下是哪个`hwmonx`，这里重新启动后，驱动正常加载，hwmon号变为了2，也即是现在变为了`/sys/class/hwmon/hwmon2`是控制风扇转了。

知道这些信息后，我们还可以进一步针对不同的风扇，测量其启动转速，这个根据风扇的不同略有不同。

自此，调速的前置条件我们都满足了，接下来就可以写一个简单的调速脚本来做这个事了，为了方便各位，皮蛋熊在这里已经写好了，开源并打包为了一个DEB包。 [gpufan-ctrl](https://github.com/pdbear/gpufan-ctrl)
安装：
```bash
dpkg -i gpufan_ctrl-amd64.deb
```
修改配置文件：
```bash
# vim /etc/gpufan-ctrl/config.json
{
    "interval"  : "1",  # 调控间隔
    "hwmon"     : "4",  # hwmon号，根据上面的教程查询
    "pwm_chan"  : "4",  # 控制风扇通道
    "fan_chan"  : "4",  # 读取风扇转速通道，某些主板这里可能不是和pwm通道对应
    "fan_min"   : "25", # 开始风扇控制时候pwm占空比
    "fan_max"   : "70", # pwm占空比最高值
    "tmp_start" : "45", # gpu温度超过这个值过后开始调控转速
    "tmp_ratio" : "2",  # 每升高1° pwm占比空升高比例
    "tmp_high"  : "82"  # gpu温度超过这个值将全速运转
}
```
配置文件修改完成后，可以试试输入:
```bash
gpufan-ctrl
```
等待输出：
```bash
root@pve:~# gpufan-ctrl
| GPU Temp | Fan Speed | Fan RPM |
|      49° |       33% |    2758 |
|      49° |       33% |    2727 |
|      49° |       33% |    2603 |
|      49° |       33% |    2603 |
|      48° |       31% |    2614 |
|      48° |       31% |    2521 |
|      48° |       31% |    2448 |
|      47° |       29% |    2448 |
|      47° |       29% |    2343 |
|      47° |       29% |    2316 |
|      47° |       29% |    2298 |
|      46° |       27% |    2294 |
|      46° |       27% |    2307 |
|      46° |       27% |    2162 |
```
可以看到随着GPU温度降低，风扇转速逐步降低。
此时我们输入以下指令，即可实现开机自动启动调速：
```bash
systemctl start gpufan-ctrl.service
```

折腾之路并不会一帆风顺，或许你会遇到这样那样的问题，也或许你有更完美的方案，Q群：573116017 欢迎你到来一起探讨。