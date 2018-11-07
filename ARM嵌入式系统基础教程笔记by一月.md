### 4.8 GPIO

​									GPIO与控制寄存器的关系：

![1538382038005.png](https://i.loli.net/2018/10/04/5bb5f95a2d23b.png)



将第 i 位置1：|=(1<<i);

将第 i 位置0：`&=~(1<<i);`

#### IOxDIR : GPIO方向寄存器

当引脚为GPIO模式时，配置引脚为输入/输出方向。0=输入，1=输出。`

- 例：将p0.3、P0.7配置为输入，p0.1、P0.5为输出。

  `IO0DIR &= ~((1<<7)|(1<<3));`

  `IO0DIR |= (1<<1)|(1<<5);`

- 例：将p0.6、p0.7配置为输出。

  `IO0DIR |=(3<<6);//若其他管脚没有使用，可用IO0DIR = c0;`

#### IOxSET : GPIO输出置位寄存器

将某些位写入1，使其对应管脚输出高电平。

#### IOxCLR : GPIO输出清零寄存器

将某些位写入1，使其对应管脚输出低电平。

#### IOxPIN :  GPIO引脚值寄存器

反应引脚当前电平值，只读。

#### 总结例题：

##### GPIO寄存器配置：p0.0输出高电平



![1538382905180.png](https://i.loli.net/2018/10/04/5bb5f95a2fa47.png)

```c
PINSEL0 = 0x00;//配置为GPIO口

IO0DIR |= 0x01；//配置为输出

IO0SET=0X01;//p0.0输出高电平	
```

##### 蜂鸣器控制电路：让蜂鸣器时响时不响：


![1538386683061.png](https://i.loli.net/2018/10/04/5bb5f95a4b5f2.png)


```c
//延时函数
void DelayNS(uint32 dly){
	uint i;
	for(;dly>0;dly--){
		for(i=0;i<5000;i++){}
	}
}
int main(){
	PINSEL0 = 0X00; //配置p0.7为GPIO口
	IO0DIR |= (1<<7); // IO0DIR = 0X80; 设置I/O为输出
	while(1){
		IO0SET = 1<<7;  // IO0SET = 0x80; 蜂鸣器灭
		DelayNS(20);
		IO0CLR = 1<<7; // IO0CLR = 0x80; 蜂鸣器响
		DelayNS(20); 
	}
	return 0;
}
```

##### 键控开关-蜂鸣器开关电路(用查询的办法,中断办法看4.9的总结例题)



![1538408154150.png](https://i.loli.net/2018/10/04/5bb5f95a4fb11.png)



功能描述：按键控制蜂鸣器，奇数次响，偶数次不响。

```c
#define BUZZ 1<<7
#define KEY 1<<20
#define uint32  unsigned int
void main(void){
	uint32 flag=0,key_status;/*flag--标记蜂鸣器状态，
							   key_status--记录按键电平值*/
	PINSEL0 = 0X00; //配置p0.7为GPIO口
	PINSEL1 = 0X00; //配置p0.20为GPIO口
	IO0DIR |= BUZZ; //设置P0.7为输出
	IO0DIR &=~KEY; //设置p0.20为输入
	while(1){
		key_status = IO0PIN&KEY;
		if(key_status == 0){
			DelayNS(10);	//按键去抖，去噪
			key_status = IO0PIN&KEY;
			if(key_status == 0){  		
			//写法1：
				flag = (flag == 0)? 1:0; /* if(flag == 0) flag = 1;
											else flag = 0; */
				if(flag ==0)
					IO0CLR = BUZZ; //蜂鸣器响
				else
					IO0SET = BUZZ; //蜂鸣器不响		
			/*		
			写法2：
				if(flag == 0)
					IO0CLR = BUZZ;
				else 
					IO0SET = BUZZ;
				flag=~flag;
			*/
              //若要蜂鸣器响一会不响一会，则在IO0CLR = BUZZ；和IO0SET = BUZZ；后面均加上
              //  DelayNS(time);
			}
		}
	}
}
```

### 4.9 向量中断控制器

![1538559072151.png](https://i.loli.net/2018/10/04/5bb5fa0013340.png)

|        中断类型         | 优先级         |
| :---------------------: | -------------- |
| FIQ中断（快速中断请求） | 具有最高优先级 |
|       向量IRQ中断       | 具有中等优先级 |
|      非向量IRQ中断      | 具有最低优先级 |

20个中断请求输入，12个未使用。16个向量IRQ中断。中断源看课本p190-191。

#### 1 VIC控制寄存器：

##### ①选择产生中断的类型：

###### VICIntSelect(中断选择寄存器)：

32bit。第某位写入1时，对应中断源产生的中断为FIQ。写入0，则为IRQ中断。

##### ②允许中断源产生中断：

###### VICIntEnable(中断使能寄存器)：

32bit。第某位写入1时，允许对应的中断源产生中断。

使用：

`VICIntEnable = 1<<14;//使能EINT0中断`

###### VICIntEnClear(中断使能清零寄存器)：

32bit。与中断使能寄存器功能相反。第某位写入1时，禁止对应的中断源产生中断。

#### 2 VIC参数设置寄存器：

##### ①向量IRQ中断相关寄存器：

###### VICVectCntlx(x=0~15)(向量控制寄存器)：

32bit。向量IRQ通道0的优先级最高，向量IRQ通道15的优先级最低。

|  位  | 31 : 6 |        5        |   [4:0]    |
| :--: | :----: | :-------------: | :--------: |
| 功能 |  保留  | EN(向量IRQ使能) | 中断源序号 |

   使用：

`VICVectCntlx = 0x20|中断源序号;  //VICVectCntlx = (1<<5)|中断源序号` (x=0~15)

###### VICVectAddrx(向量地址寄存器0~15)：

32bit。该寄存器中存放对应优先级向量IRQ中断服务程序的入口地址。

使用：

假设中断服务程序地址为 Timer0_ISR。

```c
void __irq Timer0_ISR(void)
{
	{中断处理}
	T0IR = 0X01；//清楚定时器0的标志
	VICVectAddr = 0x00;//通知VIC中断处理结束
}
...
VICVectAddrx = (unsigned int)Timer0_ISR //就是那个中断函数名啦
```

##### ②非向量IRQ中断相关寄存器：

###### VICDefVectAddr(默认向量地址寄存器)：

用法同VICVectAddrx。

##### ③产生中断后的服务程序地址：

###### VICVectAddr(向量地址寄存器)：

用于存放IRQ中断服务程序地址。当发生一个IRQ中断后，CPU读取该寄存器并跳转到对应地址处，执行中断服务程序。不能直接赋值给它。该寄存器的值从VICVectAddr0~15或VICDefVectAddr中复制得到。

注意：需要在中断服务程序最后将其清零。

`VICVectAddr = 0x00;`

#### 3 状态寄存器(均为32bit)

具体看书p198。

##### ①VICIRQStatus(IRQ状态寄存器):

读取该寄存器可确定哪些中断被激活。当某位为1时表示对应位的中断源产生IRQ中断请求。

##### ②VICFIQStatus(FIQ状态请求寄存器):

当某位为1时表示对应位的中断源产生FIQ中断请求。

##### ③VICRawIntr(所有中断的状态寄存器)：

当某位为1时表示对应位的中断源产生中断请求。

#### 4.FIQ中断(课本p200-204)

##### FIQ中断使能函数：`FIQEnable();`

##### FIQ中断禁止函数：`FIQDisable();`

##### FIQ服务程序：

```c
void FIQ_Exception(void){
	{FIQ中断处理}
	...
}
```

##### 例：

将外部中断0(EINT0)分配为FIQ，并编写FIQ服务程序。(课本p201)

```C
//初始化 
FIQEnable(); 					//使能FIQ中断(课本没有这行，这是必须的)
VICIntSelect =(1<<EINT0_num);   //EINT0分配为FIQ中断
EXTINT = 0X01; 					//清除EINT0中断标志(详细说明见外部中断)
VICIntEnable = (1<<EINT0_num);  //使能EINT0中断

//FIQ服务程序
void FIQ_Exception(void){
	{FIQ中断处理}
	while((EXTINT&0X01)!=0)
    {
        EXTINT = 0X01; 			//清除EINT0中断标志
    }
}
```

#### 5.向量IRQ中断(课本p204-210)

##### IRQ中断使能函数：`IRQEnable();`

##### IRQ中断禁止函数：`IRQDisable();`

##### IRQ服务程序：

```c
void __irq xxx(void){ //xxx为中断服务程序地址，即函数名
    {中断处理}
    ...
    VICVectAddr = 0x00; //通知VIC中断处理结束
}
```

##### 例：

将定时器0(Timer0)分配为向量IRQ通道0，中断服务程序地址设置为Timer0_ISR。(课本p206-207)

```c
//初始化 
IRQEnable();							//使能IRQ(课本没有这行，这是必须的)
VICIntSelect = 0x00;  					//所有中断通道均设为IRQ中断
VICVectCntl0 = 0x20|Timer0_num;			//等价于VICVectCntl0 = (1<<5)|Timer0_num; 
//↑将定时器0分配为通道0
VICVectAddr0 = (unsigned int)Timer0_ISR;//清除Timer0中断服务程序地址
T0IR = 0x01;							//清除Timer0中断标志
VICIntEnable = (1<<Timer0_num);			//使能Timer0中断

//IRQ服务程序
void __irq Timer0_ISR(void){
    {中断处理}
    T0IR = 0x01；						//清除中断标志
    VICVectAddr = 0x00;					//通知VIC中断处理结束
}

```

#### 6.非向量IRQ中断(课本p210-212)

在向量IRQ够用的情况下，最好不要使用非向量IRQ中断。

也要用到IRQ中断使能函数。若要将向量IRQ中断改为非向量IRQ中断，只需把**VICVectCntlx**和**VICVectAddrx**两行代码改为**VICDefVectAddr = (unsigned int)中断服务程序地址**。

##### 例：

将外部中断0(EINT0)分配为非向量IRQ，并编写中断服务程序。(课本p211)

```c
//初始化
IRQEnable();							 //使能IRQ(课本没有这行，这是必须的)
VICIntSelect = 0x00;					 //设置所有中断为IRQ中断
VICDefVectAddr = (unsigned int)Eint0_ISR;//设置中断服务程序地址
EXTINT = 0X01;							 //清除EINT0中断标志
VICIntEnable = (1<<EINT0_num);			 //使能EINT0中断

//中断服务程序
void __irq Eint0_ISR(void){
    {中断处理}
    while((EXTINT&0X01)!=0)
    {
        EXTINT = 0X01; 			//清除EINT0中断标志
    }
    VICVectAddr = 0; 			//向量中断结束
}
```

#### 总结例题：

##### 用中断的方法实现4.8的总结例题。

###### ①用FIQ中断实现：

```c
#define BUZZ 1<<7

int main(){
    int flag=0;				//标志蜂鸣器
    FIQEnable();			//使能FIQ中断
    PINSEL0 = 0x00;			//配置p0.7为GPIO口
    PINSEL1 = 3<<8;			//配置P0.20为EINT3
    IO0DIR |= BUZZ;			//配置p0.7为输出
    VICIntSelect = 1<<17;	//使能EINT3中断
    EXTMODE = 0X00;			//EINT3中断为电平触发模式
    EXTPOLAR = 0X00;		//EIN3中断为低电平触发
    EXTINT = 1<<3;			//清除EINT3中断标志
    VICIntEnable = 1<<17;	//允许EINT3中断
    while(1);				//等待中断
    return 0;
}
void FIQ_Exception(void){
    flag = (flag == 0)?1:0;//不懂看4.8总结例题注释
    if(flag == 0)
        IO0CLR = BUZZ;
    else
        IO0SET = BUZZ;
    EXTINT = 1<<3;			//清除EINT3中断标志
    VICVectAddr = 0;		//这是硬件决定的，有些要清0有些不用，可不写
}
```

###### ②用向量IRQ中断实现：

```c
void main(void){
    int flag=0;						//标志蜂鸣器
    IRQEnable();					//使能IRQ中断
    PINSEL0 = 0x00;					//配置p0.7为GPIO口
    PINSEL1 = 3<<8;					//配置P0.20为EINT3
    IO0DIR |= BUZZ;					//配置p0.7为输出
    VICIntSelect = 0x00;			//所有中断分配为IRQ中断
    VICVectCntl0 = (1<<5)|17;		//EINT3分配为通道0
    VICVectAddr0 = (int)IRQ_Eint3;	//中断服务程序地址
    EXTMODE = 0X00;					//EINT3中断为电平触发模式
    EXTPOLAR = 0X00;				//EIN3中断为低电平触发
    EXTINT = 1<<3;					//清除EINT3中断标志
    VICIntEnable = 1<<17;			//允许EINT3中断
    while(1);						//等待中断
}

void __irq IRQ_Eint3(void){
    flag = (flag == 0)?1:0;
    if(flag == 0)
        IO0CLR = BUZZ;
    else
        IO0SET = BUZZ;
    EXTINT = 1<<3;					//清除EINT3中断标志
    VICVectAddr = 0; 				//必须的，清0，向量中断结束
}
```

###### ③用非向量IRQ中断实现：

将向量IRQ里

```c
VICVectCntl0 = (1<<5)|17;		//EINT3分配为通道0
VICVectAddr0 = (int)IRQ_Eint3;	//中断服务程序地址
```

这两句换成`VICDefVectAddr = (int)IRQ_Eint3; //中断服务程序地址`

完整版：

```c
void main(void){
    int flag=0;						//标志蜂鸣器
    IRQEnable();					//使能IRQ中断
    PINSEL0 = 0x00;					//配置p0.7为GPIO口
    PINSEL1 = 3<<8;					//配置P0.20为EINT3
    IO0DIR |= BUZZ;					//配置p0.7为输出
    VICIntSelect = 0x00;			//所有中断分配为IRQ中断
    
    VICDefVectAddr = (int)IRQ_Eint3; //中断服务程序地址
    
    EXTMODE = 0X00;					//EINT3中断为电平触发模式
    EXTPOLAR = 0X00;				//EIN3中断为低电平触发
    EXTINT = 1<<3;					//清除EINT3中断标志
    VICIntEnable = 1<<17;			//允许EINT3中断
    while(1);						//等待中断
}

void __irq IRQ_Eint3(void){
    flag = (flag == 0)?1:0;
    if(flag == 0)
        IO0CLR = BUZZ;
    else
        IO0SET = BUZZ;
    EXTINT = 1<<3;					//清除EINT3中断标志
    VICVectAddr = 0; 				//必须的，清0，向量中断结束
}
```

