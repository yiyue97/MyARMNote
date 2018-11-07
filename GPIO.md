### 目录

<!-- TOC -->

- [目录](#目录)
- [基本位运算](#基本位运算)
- [GPIO与控制寄存器的关系](#gpio与控制寄存器的关系)
- [GPIO 寄存器](#gpio-寄存器)
	- [IOxDIR : GPIO方向寄存器](#ioxdir--gpio方向寄存器)
	- [IOxSET : GPIO输出置位寄存器](#ioxset--gpio输出置位寄存器)
	- [IOxCLR : GPIO输出清零寄存器](#ioxclr--gpio输出清零寄存器)
	- [IOxPIN :  GPIO引脚值寄存器](#ioxpin---gpio引脚值寄存器)
- [例题](#例题)
	- [GPIO寄存器配置：P0.0输出高电平](#gpio寄存器配置p00输出高电平)
	- [蜂鸣器控制电路：让蜂鸣器时响时不响](#蜂鸣器控制电路让蜂鸣器时响时不响)
	- [键控开关 - 蜂鸣器开关电路](#键控开关---蜂鸣器开关电路)

<!-- /TOC -->

### 基本位运算

> 将第 `i` 位置为  `1`   -> `PINSEL1 |= (1 << i);`
>
> 将第 `i` 位置为 `0`  -> `PINSEL1 &= ~(1 << i);`



### GPIO与控制寄存器的关系

![1538382038005.png](https://i.loli.net/2018/10/04/5bb5f95a2d23b.png)



### GPIO 寄存器

#### IOxDIR : GPIO方向寄存器

当引脚为GPIO模式时，配置引脚为输入/输出方向。0 == 输入，1 == 输出。

- 例：将 P0.3、P0.7 配置为输入，P0.1、P0.5为输出。

	`IO0DIR &= ~((1 << 7) | (1 << 3));`

	`IO0DIR |= (1 << 1) | (1 << 5);`

- 例：将p0.6、p0.7配置为输出。

	`IO0DIR |= (3 << 6); //若其他管脚没有使用，可用IO0DIR = 0xC0;`

#### IOxSET : GPIO输出置位寄存器

将某些位写入 1，使其对应管脚输出高电平。

#### IOxCLR : GPIO输出清零寄存器

将某些位写入 1，使其对应管脚输出低电平。

#### IOxPIN :  GPIO引脚值寄存器

反应引脚当前电平值，只读。





### 例题

#### GPIO寄存器配置：P0.0输出高电平



![1538382905180.png](https://i.loli.net/2018/10/04/5bb5f95a2fa47.png)

```c
PINSEL0 = 0x00; // 配置为GPIO口
IO0DIR |= 0x01;// 配置为输出
IO0SET=0X01; // p0.0输出高电平	
```





#### 蜂鸣器控制电路：让蜂鸣器时响时不响

![1538386683061.png](https://i.loli.net/2018/10/04/5bb5f95a4b5f2.png)

```c
//延时函数
void DelayNS(uint32 dly)
{
	uint i;
	for(; dly>0; dly--) {
		for(i=0; i<5000; i++) {}
	}
}

int main(void)
{
	PINSEL0 = 0x00; //配置p0.7为GPIO口
	IO0DIR |= (1 << 7); // IO0DIR = 0X80; 设置I/O为输出
	while (1) {
		IO0SET = 1 << 7;  // IO0SET = 0x80; 蜂鸣器灭
		DelayNS(20);
		IO0CLR = 1 << 7; // IO0CLR = 0x80; 蜂鸣器响
		DelayNS(20);
	}
	return 0;
}
```





#### 键控开关 - 蜂鸣器开关电路




![1538408154150.png](https://i.loli.net/2018/10/04/5bb5f95a4fb11.png)



用查询的方式，功能描述：按键控制蜂鸣器，奇数次响，偶数次不响。

```c

#define BUZZ 1<<7
#define KEY 1<<20
#define uint32  unsigned int

int main(void)
{
	// flag 标记蜂鸣器状态
	// key_statu 记录按键电平值
	char flag = 0;
	uint32 key_status;

	PINSEL0 = 0x00; // 配置p0.7为GPIO口
	PINSEL1 = 0X00; // 配置p0.20为GPIO口
	IO0DIR |= BUZZ; // 设置P0.7为输出
	IO0DIR &= ~KEY; // 设置p0.20为输入

	while (1) {
		key_status = IO0PIN & KEY;
		if (key_status == 0) {
			DelayNS(10);	//按键去抖，去噪
			key_status = IO0PIN & KEY;
			if (key_status == 0) {
				flag = (flag == 0) ? 1 : 0;
				if (flag ==0)
					IO0CLR = BUZZ; //蜂鸣器响
				else
					IO0SET = BUZZ; //蜂鸣器不响
#if 0
				if (flag == 0)
					IO0CLR = BUZZ;
				else
					IO0SET = BUZZ;
				flag = ~flag;
#endif
				/**
				 * 若要蜂鸣器响一会不响一会
				 * 则在IO0CLR = BUZZ; 
				 * 和 IO0SET = BUZZ;
				 * 后面均加上 DelayNS(time);
				 */
			}
		}
	}
	return 0;
}
```