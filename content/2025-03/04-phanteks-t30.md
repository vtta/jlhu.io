装上了T30, 这个扇子更厚, 风压更好, 对厚冷排更友好. 换这个是因为我们的ES CPU有bug, 在60+度就有降频现象. T30的最高转速相比之前的A12x25的1500RPM能高一倍到3000RPM. 风量也会更好. 噪音的话需要尝试调速来解决.

关于supermicro X12: BMC中只能调模式, 不能细节控制转速. 手动条件需要细致的了解.

目前已知BMC中将FAN分为了两个区, CPU区(ID为0, 包括FAN1-6)以及HDD区(ID为1, 包括FANA-B). 想控制某个区的转速可以通过:

```shell
sudo ipmitool raw 0x30 0x70 0x66 0x01 <zone> <percentage>
# FAN1: radiator x2
# FAN2: radiator x3
# FAN5: case     x1
# 70%=1960RPM 75%=2100RPM 80%=2240RPM 85%=2380RPM 90%=2520RPM 95%=2800RPM 100%=2940RPM
# FANA: pump
```

我们将所有的风扇都接入了CPU区, 水泵在HDD区. 水泵噪音基本听不见, 使用heavy IO模式正好可以让水泵满转. 我们只需要管理风扇转速. T30调到75%正好是2100转, 噪音可以接受. 即使用`sudo ipmitool raw 0x30 0x70 0x66 0x01 0 75`命令. 

手动控制的问题是有效期似乎只有几分钟, 之后会被重置会BMC设定的值. 可以使用[`smfc`](https://github.com/petersulyok/smfc)在后台接管. 