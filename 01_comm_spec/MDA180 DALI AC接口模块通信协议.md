# MDA180 DALI AC 接口模块通信协议

版本：v1.0， 更新日期：2022-10-18

 2022(c) 南京美加杰智能科技有限公司 www.meijay.com



本文档定义了MDA180 DALI **ACIP** （**A**pplication **C**ontroller **I**nterface **P**rotocal，应用控制器模块接口通信协议)。

## 接口 

### UART

#### 硬件接口

硬件接口默认为通用异步串行口(UART)，3.3V TTL逻辑电平。如果通过RS232收发器转换为RS232接口，参数和传输时序均保持不变。

引脚包括：

* **MDA_TXD**：模块发送。
* **MDA_RXD**：模块接收。
* **MDA_NRST**： 模块MCU复位，低电平复位，高电平释放。

#### 参数设置

UART通信默认参数设置为：**115200bps**, **8**-**N**-**1**。 每个字节的传输时间约为86.8us。

#### 传输时序
* 数据帧内字节传输间隔  < 1.5个字节传输长度 =~130us。
* 数据帧之间的静默时间  > 3.5个字节传输长度 =~300us。

#### 数据帧格式

**数据帧**的长度为 6~255 字节， 组成为：

| 1 byte | 3~251 bytes | 1 byte |
| :----: | :---------: | :----: |
|  SOF   |     PDU     |  FCS   |

其中：

- **SOF**:  **S**tart **O**f **F**rame，帧起始标志，等于0xFE，用于帧同步。
- **PDU**: PDU（**P**rotocol **D**ata **U**nit，协议数据单元），格式含义根据FrameType不同分别定义。
- **FCS**:  **F**rame **C**heck **S**ignature，帧校验标志，PDU部分内容按字节 XOR 校验和 。



**FCS计算示例**

```c
unsigned char calcFCS(unsigned char *pMsg, unsigned char len)
{
	unsigned char result = 0;
	while (len--)
	{
		result ^= *pMsg++;
	}
	return result;
}
```



### RS485

#### 硬件接口

* MDA_TXD：连接至RS485收发器的发送信号引脚

* MDA_RXD：连接至RS485收发器的接收信号引脚

* MDA_DE：连接至RS485收发器的发送使能引脚

#### 参数设置

RS485通信默认参数设置为：**19200bps**（或**38400bps**）, **8**-**N**-**1**。 每个字节的传输时间约为520.8（或260.4us）。

#### 传输时序

* 数据帧内字节传输间隔  < 1.5个字节传输长度 =~781us（或390.5us）。
* 数据帧之间的静默时间  > 3.5个字节传输长度 =~1823us（或911.5us）。

#### 数据帧格式

**数据帧**的长度为 6~255 字节， 组成为：

| 1 byte |  1 byte  | 3~251 bytes | 2 byte |
| :----: | :------: | :---------: | :----: |
|  SOF   | SiteAddr |     PDU     |  FCS   |

其中：

- **SOF**:  **S**tart **O**f **F**rame，帧起始标志，1字节，等于0xFE，用于帧同步。（备注：如果使用了数据帧间的时序要求，起始字节似乎没有必要）。
- **SiteAddr**： 从站地址，1字节，取值范围0~255，使用RS485接口时表示从站地址，0表示广播。
- **PDU**: PDU（**P**rotocol **D**ata **U**nit，协议数据单元）， 格式含义根据FrameType不同分别定义。
- **FCS**:  2 字节，SiteAddr和PDU部分内容按字节计算CRC-16，多项式为 (1 + x^2 + x^15 + x^16)，和Modbus RTU的计算方式一致，但CRC结果的高字节在前，低字节在后。

## PDU 协议数据单元

### PDU格式

PDU的组成如下：

| 1 byte |    1 byte    | 1 byte | 0~249 byte(s) |
| :----: | :----------: | :----: | :-----------: |
| Length | FrameControl | CmdId  |     Data      |

其中：

- **Length**：Data 长度， 1字节，取值范围 0~249。
- **FrameControl**： 控制，1字节，详见 [FrameControl 字节](#FrameControl 字节)。
- **CmdId**：命令标识， 1字节，取值范围0~255。
- **Data**: 有效数据，0~249字节。

#### FrameControl 字节

|   Bit 7   |  bit 6:4  | bit 3:0 |
| :-------: | :-------: | :-----: |
| Direction | FrameType | TrackID |

其中：

- bit[7]：**Direction**， 方向。0：主机发送请求；1：模块发送响应或报告；
- bit[6:4]：**FrameType**， 帧类型，取值0~7。
- bit[3:0]：**TrackID**, 追溯号，取值范围0~15。一般由主机指定， 通常应采取在1~15循环递增的策略，模块返回与请求帧相关的响应帧应使用相同的TrackID以便主机追溯。当模块无法确定TrackID或者模块主动发起数据帧传输时取值0。

### 帧类型列表

FrameType指示数据帧的类型。

| FrameType | 名称             | 说明                                                         |
| --------- | ---------------- | ------------------------------------------------------------ |
| 0         | **Poll**         | 轮询获取队列消息（仅限于RS485和SPI接口），一般用于主机发送给模块 |
| 1         | **SyncRequest**  | 同步请求，期望获得SyncResponse，一般用于主机发送给模块       |
| 2         | **AsyncRequest** | 异步请求，主机发送给模块                                     |
| 3         | **SyncResponse** | 同步响应，接收到SyncRequest后立即发出的响应，一般用于模块发送给主机 |
| 4         | **AsyncReport**  | 异步汇报，一般用于模块发送给主机                             |
| 5         | **Reserved**     | 保留未用                                                     |
| 6         | **AckNak**       | 应答，包括ACK和NACK。                                        |
| 7         | **Exception**    | 异常                                                         |

### 应答帧（AckNack）

应答帧仅用于模块确认接收到主机的请求帧。

* ACK表示数据帧接收正常。

* NACK表示数据帧错误或者模块内部接收缓冲区满无法处理，主机需要检查错误或者等待重传。

应答帧的PDU部分内容定义如下：

| 字段           | 数值             | 说明                      |
| -------------- | ---------------- | ------------------------- |
| Length         | 0              |  |
| FrameControl | 0b1110xxxx | Direction=1； FrameType=6；FrameSeq: ACK时和主机请求帧相同，NACK时为15。 |
| CmdId  |  | 表示AckNack的类型。0：ACK；1~255：NACK，参考NACK Code。 |
| Data           |      | 无。 |

**NACK 代码 **

| NAK Code | 名称               | 含义                                      |
| -------- | ------------------ | ----------------------------------------- |
| 1        | Illegal Frame      | 非法数据帧，FrameType不是模块支持的类型。 |
| 2        | Module Buffer Full | 模块接收缓冲区满，无法接收数据帧。        |
| 3        | Module Not Ready   | 模块未就绪，无法处理所请求的命令。        |
| 4        | UnsupportedCmdId   | 不支持的CmdId。                           |
| 5~255    | Reserved           | 保留未用。                                |



### 异常帧（Exception Frame）

异常帧用于模块向主机汇报数据请求执行过程中的错误，异常代码存放在异常帧的Data中。

> 异常帧和NACK帧是有区别的，异常帧建立在主机请求已经ACK后在执行过程中出现的错误。NACK帧是模块在数据接收阶段检测到数据帧错误或模块无法处理。

PDU定义如下：

| 字段         | 数值       | 说明                                                    |
| ------------ | ---------- | ------------------------------------------------------- |
| Length       | 1          |                                                         |
| FrameControl | 0b1111xxxx | Direction=1; FrameType=7; FrameSeq和主机请求帧相同。    |
| CmdId        |            | 和请求帧相同。                                          |
| Data         | ErrorCode  | 数据长度为1字节，表示错误代码，含义详见“错误代码”列表。 |

**错误代码列表**

| ErrorCode | 名称                   | 含义                                                         |
| --------- | ---------------------- | ------------------------------------------------------------ |
| 1         | Illegal Command        | 非法命令，CmdCat和CmdId组合不是模块支持的命令功能。          |
| 2         | Illegal Data           | 非法数据，数据帧携带的Data不是合法数据。                     |
| 3         | Module Service Busy    | 模块服务忙，无法处理所请求的命令。                           |
| 4         | Module Service Failure | 模块服务失效，执行请求命令失败。                             |
| 5         |                        | TODO：增加每一个接口大类的错误代码，为请求执行错误进行统一处理。 |

## 通信流程

### 主机发送请求

一般情况下，通信均由主机发起，主机向模块发送请求，模块响应主机的请求。

模块响应可能包含多个响应数据帧，如下：

* 模块接收到主机请求后，检查数据帧是否合法并立即回复应答帧（AckNack），此应答中Data部分只包含和主机请求帧中相同的FrameSeq， 用于确认主机请求帧的正确接收。如模块在执行请求时出现错误，则返回异常帧（Exception）。
* 若该请求为异步操作（如发送DALI透传命令或执行DALI应用命令），则模块在DALI发送完成后根据主机的请求中选项标志可以向主机发送汇报数据帧。
* 若该请求为包含多个步骤的异步操作（如DALI透传查询命令或DALI应用命令），则模块在异步操作的每一步状态变化时（例如收到DALI总线上的控制装置响应数据）都可以向主机发送汇报数据帧。

多个响应之间的超时控制请参考每个数据帧的定义说明。

### 模块主动上报

模块在以下几种情况下会向主机主动发起上报：

* 执行耗时较长的异步操作过程中状态发生了变化，或操作完成时。
* 检测到DALI总线状态变化或检测到了数据。
* 定时任务。

> 注意： 若为RS485应用，应禁用模块主动上报功能，而由主机采用轮询的方式来读取模块上报的消息。

### 数据重传

当接收方接收的数据出现以下问题时：

* 数据帧错误：FCS校验错误、Control字节非法和对应帧类型的Length非法
* 模块缓冲区满或未就绪

## 接口命令定义

| 名称                | CmdId | Dir | FrameType    | 描述             | 相关响应/请求          |
| ------------------- | ----- | --------- | ------------ | ---------------- | ---------------------- |
| **SYS_RESET_REQ**   | 0x00  | 0         | SyncRequest  | 请求模块复位     | AckNack，SYS_RESET_IND |
| **SYS_RESET_IND**   | 0x80  | 1         | AsyncReport  | 模块复位指示     | SYS_RESET_REQ |
| **SYS_VERSION**     | 0x01  | 0         | SyncRequest  | 获取模块版本信息 | SYS_VERSION_RSP        |
| **SYS_VERSION_RSP** | 0x81 | 1         | SyncResponse | 模块版本信息响应 | SYS_VERSION |
| **SYS_CFG_READ** | 0x02  | 0         | SyncRequest  | 读取模块配置数据 | SYS_CFG_READ_RSP |
| **SYS_CFG_READ_RSP** | 0x82 | 1 | SyncResponse | 模块配置读取响应 | SYS_CFG_READ |
| **SYS_CFG_WRITE** | 0x03 | 0         | SyncRequest  | 写入模块配置数据 | SYS_CFG_WRITE_RSP |
| **SYS_CFG_WRITE_RSP** | 0x83 | 1 | SyncResponse | 模块配置写入响应 | SYS_CFG_WRITE |
| **DACM_INFO**               | 0x10 | 0         | SyncRequest  | 获取DALI信息                         | DACM_INFO_RSP           |
| **DACM_INFO_RSP** | 0x90 | 1 | SyncResponse | DALI信息响应 | DACM_INFO |
| **DACM_START**              | 0x11 | 0         | SyncRequest  | 开启DALI通道                         | AckNack                 |
| **DACM_STOP**               | 0x12 | 0         | SyncRequest  | 关闭DALI通道                         | AckNack                 |
| **DACM_STATUS**             | 0x13 | 0         | SyncRequest  | 获取DALI通道状态                     | DACM_STATUS_RSP         |
| **DACM_STATUS_RSP** | 0x93 | 1 | SyncResponse | DALI通道状态响应 | DACM_STATUS |
| **DACM_CONTACT_ON**         | 0x14 | 0         | SyncRequest  | 打开DALI通道对应的回路干接点信号     | AckNack                 |
| **DACM_CONTACT_OFF**        | 0x15 | 0         | SyncRequest  | 关闭DALI通道对应的回路干接点信号     | AckNack                 |
| **DACM_CONTACT_STATUS**     | 0x16 | 0         | SyncRequest  | 获取DALI通道对应的回路干接点信号状态 | DACM_CONTACT_STATUS_RSP |
| **DACM_CONTACT_STATUS_RSP** | 0x96 | 1         | SyncResponse | DALI通道对应的回路干接点信号状态响应 | DACM_CONTACT_STATUS |
| **DACM_BPS_CTRL** | 0x17 | 0 | SyncRequest | DALI总线电源控制 | AckNack |
| **DACM_BPS_STATUS** | 0x18 | 0 | SyncRequest | DALI总线电源状态获取 | DACM_BPS_STATUS_RSP |
| **DACM_BPS_STATUS_RSP** | 0x98 | 1 | SyncResponse | DALI总线电源状态获取响应 | DACM_BPS_STATUS |
| **DATT_SEND**      | 0x20 | 0         | AsyncRequest | 发送DALI总线数据 | AckNack，DATT_DATA_IND |
| **DATT_SEND8**     | 0x21 | 0         | AsyncRequest | 发送8bit数据     | AckNack，DATT_DATA_IND |
| **DATT_SEND16**    | 0x22 | 0         | AsyncRequest | 发送16bit数据    | AckNack，DATT_DATA_IND |
| **DATT_SEND24**    | 0x23 | 0         | AsyncRequest | 发送24bit数据    | AckNack，DATT_DATA_IND |
| **DATT_SEND32**    | 0x24 | 0         | AsyncRequest | 发送32bit数据    | AckNack，DATT_DATA_IND |
| **DATT_RECV_POLL** | 0x29 | 0         | Poll         | 查询接收数据     | DATT_DATA_IND          |
| **DATT_DATA_IND** | 0xA9 | 1         | AsyncReport  | DALI总线数据指示 |                      |
| **DATT_EVENT_IND** | 0xAA | 1         | AsyncReport  | DALI总线事件汇报 |  |
| **DAA_BPS_CTRL**          | 0x30 | 0         | SyncRequest  | DALI总线电源控制           |AckNack|
| **DAA_BPS_STAUS**         | 0x31 | 0         | SyncRequest  | DALI总线电源状态获取       |DAA_BPS_STATUS_RSP|
| DAA_BPS_STATUS_RSP | 0xB1 | 1 | SyncResponse | DALI总线电源状态响应 |DAA_BPS_STAUS|
| **DAA_CG_DISC**           | 0x32 | 0         | AsyncRequest | DALI控制装置设备搜索       |AckNack，DAA_CG_DISC_IND|
| **DAA_CG_DISC_IND**       | 0xB2 | 1         | AsyncReport  | DALI控制装置设备搜索指示   |DAA_CG_DISC|
| **DAA_CG_ADDRESSING**     | 0x33 | 0         | AsyncRequest | DALI控制装置地址分配       |AckNack，DAA_CG_ADDRESSING_IND|
| **DAA_CG_ADDRESSING_IND** | 0xB3 | 1         | AsyncReport  | DALI控制装置地址分配指示   |DAA_CG_ADDRESSING|
| **DAA_CG_DAPC** | 0x34 | 0 | AsyncRequest | DALI控制装置电弧功率等级控制 |AckNack，DAA_CG_DAPC_CFM|
| DAA_CG_DAPC_CFM | 0xB4 | 1 | AsyncReport |  |DAA_CG_DAPC|
| **DAA_CG_CTRL**           | 0x35 | 0         | AsyncRequest | DALI控制装置控制           |AckNack，DAA_CG_CTRL_CFM|
| DAA_CG_CTRL_CFM | 0xB5 | 1 | AsyncReport |  |DAA_CG_CTRL|
| **DAA_CG_CFG**            | 0x36 | 0         | AsyncRequest | DALI控制装置配置           |AckNack，DAA_CG_CFG_CFM|
| DAA_CG_CFG_CFM | 0xB6 | 1 | AsyncReport |  |DAA_CG_CFG|
| **DAA_CG_QUERY**          | 0x37 | 0         | AsyncRequest | DALI控制装置查询           |AckNack，DAA_CG_QUERY_RSP|
| DAA_CG_QUERY_RSP | 0xB7 | 1 | AsyncReport |  |DAA_CG_QUERY|
| **DAA_CG_MB_READ**        | 0x38 | 0         | AsyncRequest | DALI控制装置MemoryBank读取 |AckNack，DAA_CG_MB_READ_RSP|
| DAA_CG_MB_READ_RSP | 0xB8 | 1 | AsyncReport |  |DAA_CG_MB_READ|
| **DAA_CG_MB_WRITE**       | 0x39 | 0         | AsyncRequest | DALI控制装置MemoryBank写入 |AckNack，DAA_CG_MB_WRITE_CFM|
| DAA_CG_MB_WRITE_CFM | 0xB9 | 1 | AsyncReport |  |DAA_CG_MB_WRITE|
| **DAA_CG_MB_RESET** | 0x3A | 0 | AsyncRequest | DALI控制装置MemoryBank复位 |AckNack，DAA_CG_MB_RESET_CFM|
| DAA_CG_MB_RESET_CFM | 0xBA | 1 | AsyncReport | |DAA_CG_MB_RESET|
| **DAA_CG_APP_CTRL**       | 0x3B | 0       | AsyncRequest | DALI控制装置扩展控制       |AckNack，DAA_CG_APP_CTRL_CFM|
| DAA_CG_APP_CTRL_CFM | 0xBB | 1 | AsyncReport |  |DAA_CG_APP_CTRL|
| **DAA_CG_APP_CFG**        | 0x3C | 0         | AsyncRequest | DALI控制装置扩展配置       |AckNack，DAA_CG_APP_CFG_CFM|
| DAA_CG_APP_CFG_CFM | 0xBC | 1 | AsyncReport |  |DAA_CG_APP_CFG|
| **DAA_CG_APP_QUERY**      | 0x3D | 0         | AsyncRequest | DALI控制装置扩展查询       |AckNack，DAA_CG_APP_QUERY_RSP|
| DAA_CG_APP_QUERY_RSP | 0xBD | 1 | AsyncReport |  |DAA_CG_APP_QUERY|
| **DAA_CG_INFO_READ** | 0x40 | 0 | AsyncRequest | DALI控制装置信息读取 |AckNack，DAA_CG_INFO_READ_RSP|
| DAA_CG_INFO_READ_RSP | 0xC0 | 1 | AsyncReport |  |DAA_CG_INFO_READ|
| **DAA_CG_VARS_READ** | 0x41 | 0 | AsyncRequest | DALI控制装置变量读取 |AckNack，DAA_CG_VARS_READ_RSP|
| DAA_CG_VARS_READ_RSP | 0xC1 | 1 | AsyncReport |  |DAA_CG_VARS_READ|
| **DAA_CG_VARS_SAVE** | 0x42 | 0 | AsyncRequest | DALI控制装置变量保存 |AckNack，DAA_CG_VARS_SAVE_CFM|
| DAA_CG_VARS_SAVE_CFM | 0xC2 | 1 | AsyncReport |  |DAA_CG_VARS_SAVE|
| **DAA_CG_APP_VARS_READ** | 0x43 | 0 | AsyncRequest | DALI控制装置应用扩展变量读取 |AckNack，DAA_CG_APP_VARS_READ_RSP|
| DAA_CG_APP_VARS_READ_RSP | 0xC3 | 1 | AsyncReport |  |DAA_CG_APP_VARS_READ|
| **DAA_CG_APP_VARS_SAVE** | 0x44 | 0 | AsyncRequest | DALI控制装置用用扩展变量保存 |AckNack，DAA_CG_APP_VARS_SAVE_CFM|
| DAA_CG_APP_VARS_SAVE_CFM | 0xC4 | 1 | AsyncReport |  |DAA_CG_APP_VARS_SAVE|
| **DAA_CG_MB_ACCESS** | 0x45 | 0 | AsyncRequest | DALI控制装置存储区访问 |AckNack，DAA_CG_MB_ACCESS_RSP|
| DAA_CG_MB_ACCESS_RSP | 0xC5 | 1 | AsyncReport |  |DAA_CG_MB_ACCESS|
| **DAA_MACRO_TEMPBUF_READ** | 0x70 | 0 | AsyncRequest | 宏指令临时缓冲区读取 |AckNack，DAA_MACRO_TEMPBUF_READ_RSP|
| DAA_MACRO_TEMPBUF_READ_RSP | 0xF0 | 1 | AsyncReport |  |DAA_MACRO_TEMPBUF_READ|
| **DAA_MACRO_TEMPBUF_WRITE** | 0x71 | 0 | AsyncRequest | 宏指令临时缓冲区写入 |AckNack，DAA_MACRO_TEMPBUS_WRITE_CFM|
| DAA_MACRO_TEMPBUS_WRITE_CFM | 0xF1 | 1 | AsyncReport |  |DAA_MACRO_TEMPBUF_WRITE|
| **DAA_MACRO_STATUS** | 0x72 | 0 | AsyncRequest | 查询宏指令运行状态 |AckNack，DAA_MACRO_STATUS_RSP|
| DAA_MACRO_STATUS_RSP | 0xF2 | 1 | AsyncReport |  |DAA_MACRO_STATUS|
| **DAA_MACRO_STOP** | 0x73 | 0 | AsyncRequest | 请求宏指令停止 |AckNack，DAA_MACRO_STOP_CFM|
| DAA_MACRO_STOP_CFM | 0xF3 | 1 | AsyncReport |  |DAA_MACRO_STOP|



### 系统管理接口

模块系统管理接口（System Management Interface）命令包括：


#### 系统复位

##### SYS_RESET_REQ

Data 内容定义如下：

| 1 byte    |
| --------- |
| ResetType |

其中，

* ResetType：1字节，复位类型。0：软件复位；1：复位到Bootloader（暂时不支持）；其他：保留未用。

回复 AckNack。

在软件复位完成后，模块主动发送 SYS_RESET_IND。

##### SYS_RESET_IND

在上电复位和接收SYS_RESET完成复位后，主动发送。

仅适用于UART或者RS232接口，对于RS485接口，不发送。



Data 内容定义如下：

| 1 byte   | 1 byte      | 1 byte   | 1 byte    | 1 byte     | 1 byte     | 1 byte     | 1 byte     | 1 byte     |
| -------- | ----------- | -------- | --------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| ResetSrc | ProtocolVer | VendorId | ProductId | FWVerMajor | FWVerMinor | FWVerPatch | HWVerMajor | HWVerMinor |

其中，

* ResetSrc： 1字节，复位源。0： 上电复位；1：外部Reset信号复位；2：看门狗复位；3：软件复位；其他：保留未用。
* ProtocolVer：1字节，模块接口协议版本。Bit[7:4]: MajorVer，0~15，主版本号；Bit[3:0]:MinorVer，0~15，次版本号。
* VendorId：1字节，厂商标识。0x00：Meijay；其他：待分配。
* ProductId：1字节，产品标识。0x00：MDA180；0x01：MDA182;其他：待分配。
* FWVerMajor：1字节，固件主版本号。
* FWVerMinor：1字节，固件次版本号。
* FWVerPatch：1字节，固件补丁版本号。
* HWVerMajor：1字节，硬件主版本号。
* HWVerMinor：1字节，硬件次版本号。



#### 系统版本和信息查询

##### SYS_VERSION

查询系统版本信息，包括协议版本、固件版本和硬件版本。

Data 内容为空。

模块回复 SYS_VERSION_RSP。

##### SYS_VERSION_RSP

Data 内容定义如下：

| 1 byte      | 1 byte     | 1 byte     | 1 byte     | 1 byte     | 1 byte     |
| ----------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| ProtocolVer | FWVerMajor | FWVerMinor | FWVerPatch | HWVerMajor | HWVerMinor |

其中，

* ProtocolVer：1字节，模块接口协议版本。Bit[7:4]: MajorVer，0~15，主版本号；Bit[3:0]:MinorVer，0~15，次版本号。
* FWVerMajor：1字节，固件主版本号。
* FWVerMinor：1字节，固件次版本号。
* FWVerPatch：1字节，固件补丁版本号。
* HWVerMajor：1字节，硬件主版本号。
* HWVerMinor：1字节，硬件次版本号。



#### 系统配置参数

配置参数分为掉电不丢失（Non-Volatile）和掉电丢失（Volatile）两种，分别用NV和VOL开头表示。

NV配置数据在模块重启复位后仍然有效，VOL配置数据需要每次复位后重新写入。

##### SYS_READ_CFG

读取配置参数。

格式：CfgId 

Data 内容定义如下：

| 1 byte    | 1 byte   |
| --------- | -------- |
| CfgIdHigh | CfgIdLow |

其中，

* CfgIdHigh：1字节，配置项目Id高字节。
* CfgIdLow：1字节，配置项目Id低字节。

回复： SYS_READ_CFG_RSP。

##### SYS_READ_CFG_RSP

Data 内容定义如下：

| 1 byte | 1 byte    | 1 byte   | 1 byte        | 1..N byte(s) |
| ------ | --------- | -------- | ------------- | ------------ |
| Status | CfgIdHigh | CfgIdLow | CfgDataLength | CfgData      |

其中，

* Status: 1字节，配置读取状态。0：Success，成功；1：InvalidCfgId，配置项Id不合法；2：CfgIdNotSupported，配置项Id不支持；其他：保留未用。
* CfgIdHigh：1字节，配置项目Id高字节。
* CfgIdLow：1字节，配置项目Id低字节。
* CfgDataLength：1字节，配置项目数据长度。
* CfgData：1~N字节，配置项目数据。



##### SYS_WRITE_CFG

写入配置参数。

Data 内容定义如下：

| 1 byte    | 1 byte   | 1 byte        | 1..N byte(s) |
| --------- | -------- | ------------- | ------------ |
| CfgIdHigh | CfgIdLow | CfgDataLength | CfgData      |

其中，

* CfgIdHigh：1字节，配置项目Id高字节。
* CfgIdLow：1字节，配置项目Id低字节。
* CfgDataLength：1字节，配置项目数据长度。
* CfgData：1~N字节，配置项目数据。

模块回复 SYS_WRITE_CFG_RSP。

##### SYS_WRITE_CFG_RSP

Data 内容定义如下：

| 1 byte | 1 byte    | 1 byte   |
| ------ | --------- | -------- |
| Status | CfgIdHigh | CfgIdLow |

其中，

* Status: 1字节，配置读取状态。0：Success，成功；1：InvalidCfgId，配置项Id不合法；2：CfgIdNotSupported，配置项Id不支持；3：InvalidDataValue，写入数据数值不合法；其他：保留未用。
* CfgIdHigh：1字节，配置项目Id高字节。
* CfgIdLow：1字节，配置项目Id低字节。

**系统参数列表**

| CfgId（2 bytes） | 数据类型（1 byte） | 名称                   | 说明                                                         |
| :--------------: | :----------------: | :--------------------- | ------------------------------------------------------------ |
|      0x0010      |        BOOL        | NV_DATT_MONITOR_ENABLE | DALI透传监控使能，开启后模块将返回所有接收到的总线数据，包括异常数据，一般用于总线监控分析。默认为false。 |
|      0x0011      |        BOOL        | NV_DATT_NO_WAIT_REPLY  | DALI透传等待响应，开启后每次透传均等待总线从机响应直到后向帧超时。默认为false。 |
|    ~~0x0012~~    |      ~~BOOL~~      | ~~NV_DATT_ECHO~~       | ~~DALI透传自发数据帧回复使能，开启后每次透传发送的数据也会返回，否则只返回命令已发送的确认帧。默认为false。~~ |
|                  |                    |                        |                                                              |



### DALI 通道管理接口
DALI 通道管理接口（DALI Channel Management Interface）命令包括：



#### DALI 通道基本管理

##### DACM_INFO

获取DALI信息，包括通道数、是否集成BPS、BPS最大电流等。

Data内容为空。

##### DACM_INFO_RSP

响应帧的Data内容定义如下：

| 1 byte           | 1 byte        | 1 byte        |
| ---------------- | ------------- | ------------- |
| NumberOfChannels | BPSIntegrated | BPSMaxCurrent |

其中：

* NumberOfChannels：DALI通道数，取值范围1~4。
* BPSIntegrated：是否集成总线电源。0：否；1：是。
* BPSMaxCurrent：集成总线电源的最大电流。0：无集成总线电源；8~250：8~250mA；其他数值：保留未用。

#### DALI 通道开关控制

##### DACM_START

开启DALI通道功能，模块将开启数据收发功能。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |



回复AckNack。

##### DACM_STOP

停止DALI通道功能，模块将关闭数据收发功能。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |



回复AckNack。

#### DALI 通道状态获取

##### DACM_STATUS

获取DALI通道状态，模块将返回DACM_STATUS_RSP。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |

##### DACM_STATUS_RSP

包括DALI数据收发开启状态、通道BPS开启状态、总线是否有故障等信息。

Data 内容定义如下：

| 1 byte  | 1 byte        | 1 byte | 1 byte     | 1 byte     |
| ------- | ------------- | ------ | ---------- | ---------- |
| Channel | TransceiverOn | BPSOn  | BPSFailure | BusFailure |

其中，

* Channel：DALI通道。
* TransceiverOn：DALI收发器开启。
* BPSOn：集成总线电源是否开启。
* BPSFailure：总线电源是否存在故障。
* BusFailure：DALI总线是否存在故障。

#### DALI 通道回路干接点控制

##### DACM_CONTACT_ON

开启回路控制节点。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |



回复AckNack。

##### DACM_CONTACT_OFF

关闭回路控制节点。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |



回复AckNack。

##### DACM_CONTACT_STATUS

获取回路控制节点状态，模块将返回DACM_CONTACT_STATUS_RSP。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |

##### DACM_CONTACT_STATUS_RSP

包括回路控制节点状态开启状态信息。

Data 内容定义如下：

| 1 byte  |  1 byte   |
| ------- | :-------: |
| Channel | ContactOn |

其中，

* Channel：通道号。
* ContactOn：回路节点是否开启。



#### DALI 总线电源管理应用接口

##### DACM_BPS_CTRL

开启DALI通道集成总线电源。

Data 内容定义如下：

| 1 byte  | 1 byte |
| :-----: | :----: |
| Channel | OnOff  |

* Channel： DALI通道。
* OnOff：开关。0：关闭BPS；1：打开BPS；其他数值：保留未用。

回复AckNack。

##### DACM_BPS_STATUS

获取DALI通道集成总线电源状态。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |

其中，

* Channel：DALI通道。

##### DACM_BPS_STATUS_RSP

包括DALI BPS 开启状态和故障信息。

Data 内容定义如下：

| 1 byte  | 1 byte | 1 byte     |
| ------- | ------ | ---------- |
| Channel | BPSOn  | BPSFailure |

其中，

* Channel：DALI通道。
* BPSOn：集成总线电源是否开启。
* BPSFailure：总线电源是否存在故障。

### DALI 透传接口
DALI 透传接口适用于主机自行实现DALI应用层管理，支持在模块DALI通道总线上收发DALI数据帧。

DALI 透传接口（DALI Transparent Transmission Interface）命令包括：

* 主机发送至模块的命令：*DATT_SEND*, *DATT_SEND8*, *DATT_SEND16*, *DATT_SEND24*, *DATT_SEND32*
* 模块发送至主机的命令：*DATT_DATA_IND*, *DATT_EVENT_IND*

主机使用透传命令的流程如下：

1. 主机发送*DATT_SENDxxx*命令；
2. 模块接收后立刻回复*AckNack*确认收到命令数据；
3. 模块解析发送请求并根据需要将发送请求分解为一个或多个需要向总线发送的数据帧并添加到数据帧发送缓冲区；
4. 模块从数据帧发送缓冲区获取数据帧，开始发送，在单个数据帧传输完成后向主机发送*DATT_DATA_IND*，指示发送情况；
5. 如果当前数据帧需要等待DALI总线上的设备回应，模块则在响应时间窗口内尝试等待接收总线数据。如果接收到数据或没有检测到响应则向主机发送*DATT_DATA_IND*指示响应接收情况；
6. 检查数据帧发送缓冲区是否为空，如果非空则跳转到第4步继续执行；
7. 如果在发送过程或等待响应过程出现总线的异常和恢复，模块均向主机发送*DATT_EVENT_IND*指示总线事件。

> **注意**：可以通过发送模块系统配置命令，禁用模块对自己发出的每个DALI数据帧时返回*DATT_DATA_IND*响应，这样可以减少中间的数据返回次数，对采用RS485通信的应用，提高了效率。但主机需要设置较长的超时等待以避免再等待多条DALI总线数据发送的过程中判断为超时。等待的时长根据最长时间估计，如同时自动设置DTR1、DTR0、DeviceType和双次发送的*DATT_SEND16*请求，模块实际需要向DALI总线发送5次16-bit数据帧，加上帧间等待（尚未考虑多主机应用的冲突重试），需要30ms*5 = 150ms。
>
> * 如果MONITOR_ENABLE开启，模块在接收到总线上的其他数据时，会向主机发送DATT_DATA_IND指示。
> * 如果ECHO开启，模块在每次向DALI总线发送数据后，会向主机发送DATT_DATA_IND指示。



#### DALI 通用透传命令

##### DATT_SEND

主机使用该命令可以向DALI总线发送任意bit长度不超过64的数据帧，含DALI目前未标准化长度的数据帧，仅用于测试，不建议作为正式的应用接口。

Data定义为：

| 1 byte  | 1 byte  | 1 byte      |   0..N byte   |
| ------- | :-----: | ----------- | :-----------: |
| Channel | Control | BusDataBits | BusData[0..N] |

其中，

* **Channel**：DALI 通道编号。
  * Bit [7:6]: 保留未用。
  * Bit [5:4]：ResponseOptions，响应选项。0：默认，按照系统参数NV_DATT_WAITREPLY_ENABLE和NV_DATT_ECHO_ENABLE的数值；1：WaitReply=0；Echo=1；2：WaitReply=1；Echo=0；3：WaitReply=1；Echo=1；
  * Bit [3:0]: Channel，取值范围0~15， DALI 通道编号。

* **Control**：控制字节。
  * Bit [7:4]：RepeatTimeInterval，两次发送的时间间隔，取值范围0~15。0：默认；1~15： 单位x10ms。
  * Bit 3： Send-Twice，发送两次。
  * Bit[2:0]：Priority，传输优先级，取值范围0~7。
* **BusDataBits**：要发送的数据位数。0：仅发送起始位；1~32：1~32位，其中8/16/24/32可以用来发送DALI标准的8/16/24/32位数据帧；33~64：不使用；65~255：无效。
* **BusData**：DALI总线上要发送的数据。高字节在前，低字节在后，根据BusDataBits从低位到高位依次发送。

##### DATT_DATA_IND

模块发出的指示总线上已经传输或接收的数据信息。

**TODO**： 合并DATT_DATA_IND 和 DATT_EVENT_IND， 通过Status来指示数据或事件类型。



Data定义为

| 1 byte  |     1 byte      |     1 byte     | 1 byte | 1 byte   | 0..N byte  |
| :-----: | :-------------: | :------------: | :----: | -------- | ---------- |
| Channel | BusIdleTimeHigh | BusIdleTimeLow | Status | DataBits | Data[0..N] |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道；1~4：通道编号；其他：保留未用。
* **BusIdleTimeHigh/BusIdleTimeLow**：  BusIdleTime的高低字节，参考[BusIdleTime](#BusIdleTime)。
* **Status**：状态， 1字节。
  * Bit[7:6]：**IndicationType**，表示数据指示的类型。0：DATA_SENT，模块向DALI总线已发出的数据；1：ANSWER_RECEIVED，从总线接收到的对应于模块发送数据的响应数据；2：DATA_RECEIVED，从DALI总线接收的其他数据；3：保留未用。
  * Bit[5:0]：**DataStatus**，已传输或接收的数据状态，指示数据是否合法及错误信息。含义参考[DATT Status](#DATT Status)说明

* **DataBits**：已传输或接收的数据位数, 1字节。已传输或接收的数据位数， 仅当Status为DATT_Success时有效。0：无数据接收；1~64：1~64位，其中8/16/24/32表示DALI标准的8/16/24/32位数据帧；65~255：无效。
* **Data**：DALI总线上接收到的数据， 0~N字节。高字节在前，低字节在后，根据DataBits从低位到高位依次接收。N = (DataBits + 7) / 8 取整。

###### BusIdleTime

BusIdleTime表示DALI总线发送或接收一个数据帧之前的空闲时间，取值范围0~0xFFFF。其中0~0xFFFE：表示时间长度为该数值 x 83.3us (即0.2Te，Te表示DALI标准中的1一个half-bit时间，约为416.7us )；0xFFFF：表示时间长度超过0xFFFE表示的值（约为5161ms，即5.16s）

###### DATT Status

| 名称                     | 数值 | 含义                                                         |
| ------------------------ | ---- | ------------------------------------------------------------ |
| DATT_Success             | 0    | 成功                                                         |
| DATT_NoReply             | 1    | 无响应                                                       |
| DATT_BusFailure          | 2    | 总线错误（可以再区分是短时间的总线错误BusError还是长时间的总线故障SystemFailure） |
| DATT_FramingError        | 3    | 数据帧时序错误（可以区分是发送冲突导致的帧错误还是接收的帧错误） |
| DATT_InvalidParameter    | 4    | 非法参数                                                     |
| DATT_TransmissionTimeout | 6    | 传输超时                                                     |

##### DATT_EVENT_IND

当总线状态发生变化时，模块发送该数据帧向主机汇报。

Data定义为

| 1 byte  |  1 byte   |
| :-----: | :-------: |
| Channel | EventType |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道；1~4：通道编号；其他：保留未用。
* **EventType**：事件类型， 1字节。含义参考[DATT EventType](#DATT EventType)说明。

###### DATT EventType

| 名称                   | 数值 | 含义                               |
| ---------------------- | ---- | ---------------------------------- |
| DATT_EVENT_NONE        | 0    | 无有效事件                         |
| DATT_EVENT_SYS_FAILURE | 1    | DALI总线掉电  (电压低> 500ms)      |
| DATT_EVENT_BUS_ERROR   | 2    | DALI总线错误（电压低 42.5ms~500ms) |
| DATT_EVENT_FRAMING_ERR | 3    | 接收数据帧错误                     |
| DATT_EVENT_BUS_OK      | 4    | DALI总线恢复正常（电压高 > 2ms )   |



#### DALI 后向帧透传

##### DATT_SEND8

在指定DALI通道所在的总线上发送8bit后向帧数据，基本不用，仅用于多个AC之间需要配置时响应其他AC。

Data定义为：

| 1 byte  | 1 byte |
| ------- | :----: |
| Channel | Answer |

其中：

* **Channel**：DALI 通道，1字节。0：所有通道；1~4：通道编号；其他：保留未用。
* **Answer**：DALI 101 中定义的8bit后向帧（Backward Frame ）响应数值， 1字节。

#### DALI 102 控制装置透传命令 

##### DATT_SEND16

在指定DALI通道所在的总线上发送16bit前向帧（Forward Frame) 命令，主要用于和控制装置（ControlGear）通信。

Data定义为：

| 1 byte  | 1 byte  | 1 byte  | 1 byte |    1 byte     |    1 byte     | 1 byte         |
| ------- | :-----: | :-----: | :----: | :-----------: | :-----------: | -------------- |
| Channel | Control | Address | Opcode | AdditionData0 | AdditionData1 | AddtionalData2 |

其中：

* **Channel**：DALI 通道，1字节。0：所有通道；1~4：通道编号；其他：保留未用。
* **Control**： 控制标志， 1字节。
* **Address**：DALI 102 控制装置地址， 1字节。数值为DALI 102 16 bit命令帧中的Address字节。
* **Opcode**：DALI 102 控制装置操作码， 1字节。数值为DALI 102 16 bit命令帧中的Opcode字节。
* **AdditionData0**：附加数据字节0， 一般用来存放SetDTR0时指定的DTR0数值。
* **AdditionData1**：附加数据字节1，一般用来存放SetDTR1时指定的DTR1数值。
* **AdditionData2**：附加数据字节2，一般用来存放SetDeviceType时指定的DeviceType数值。

**控制标志 Control**

控制标志指示透传命令的附加特性。

| bit 7           | bit 6         | bit 5   | bit 4   | bit 3      | bit [2:0] |
| --------------- | ------------- | ------- | ------- | ---------- | --------- |
| WaitForResponse | SetDeviceType | SetDTR1 | SetDTR0 | Send-Twice | Priority  |

数据位定义如下：

- bit 7：**WaitForResponse**，等待响应数据帧。0：不等待响应；1：等待响应数据。
- bit 6：**SetDeviceType**， 设置DeviceType。0：不需要设置Device Type；1：自动发送Enable Device Type x命令，Device Type数值在数据帧的AdditionData2，通常用于Device Type相关的扩展命令。
- bit 5：**SetDTR1**，设置DTR1。0： 不需要额外发送DTR1；1：自动发送DTR1命令，DTR1的数值在数据帧的AdditionData1中，通常用于需要同时指定DTR0和DTR1的命令。
- bit 4：**SetDTR0**，设置DTR0。0： 不需要额外发送DTR0；1：自动发送DTR0命令，DTR0的数值在数据帧的AdditionData0中，通常用于STORE_XXX配置命令。
- bit 3：**Send-Twice**，两次发送。0： 单次发送；1：发送2次命令（通常用于DALI命令中的Send-Twice命令）
- bit [2:0]：**Priority**, 传输优先级，默认为0，仅用于支持multi-master的应用中。0： 标准；1~5： 对应DALI标准中的Priority 0~4。6~7：保留。

#### DALI 103 控制装置透传命令

##### DATT_SEND24

在指定DALI通道所在的总线上发送24bit前向帧（Forward Frame) 命令，主要用于和DALI 103控制设备（ControlDevice）通信。

Data定义为：

| 1 byte  | 1 byte  | 1 byte  | 1 byte   | 1 byte | 1 byte        | 1 byte        | 1 byte        |
| ------- | ------- | ------- | -------- | ------ | ------------- | ------------- | ------------- |
| Channel | Control | Address | Instance | OpCode | AdditionData0 | AdditionData1 | AdditionData2 |

其中：

* **Channel**：DALI 通道，1字节。0：所有通道（默认）；1~4：通道编号；其他：保留未用。
* **Control**： 控制标志， 1字节。
* **Address**：DALI 103 控制设备地址， 1字节。数值为DALI 103 24 bit命令帧中的Address字节。
* **Instance**：DALI 103 控制设备实例，1字节。数值为DALI 103 24 bit命令帧中的Instance字节。
* **Opcode**：DALI 103 控制设备操作码，1字节。数值为DALI 103 24 bit命令帧中的Opcode字节。
* **AdditionData0**：附加数据字节0。
* **AdditionData1**：附加数据字节1。
* **AdditionData2**：附加数据字节2。

**控制标志 Control**

| bit 7           | bit 6   | bit 5   | bit 4   | bit 3      | bit [2:0] |
| --------------- | ------- | ------- | ------- | ---------- | --------- |
| WaitForResponse | SetDTR2 | SetDTR1 | SetDTR0 | Send-Twice | Priority  |

数据位定义如下：

- bit 7：**WaitForResponse**，等待响应数据帧。0：不等待响应；1：等待响应数据。
- bit 6：**SetDTR2**，设置DTR2。0： 不需要额外发送DTR2；1：自动发送DTR2命令，DTR2的数值在数据帧的AdditionData2中，通常用于需要同时指定DTR0、DTR1、DTR2或其中2个的命令。
- bit 5：**SetDTR1**，设置DTR1。0： 不需要额外发送DTR1；1：自动发送DTR1命令，DTR1的数值在数据帧的AdditionData1中，通常用于需要同时指定DTR0和DTR1的命令。
- bit 4：**SetDTR0**，设置DTR0。0： 不需要额外发送DTR0；1：自动发送DTR0命令，DTR0的数值在数据帧的AdditionData0中，通常用于STORE_XXX配置命令。
- bit 3：**Send-Twice**，两次发送。0： 单次发送；1：发送2次命令（通常用于DALI命令中的Send-Twice命令）
- bit [2:0]：**Priority**, 传输优先级，默认为0，仅用于支持multi-master的应用中。0： 标准；1~5： 对应DALI标准中的Priority 0~4。6~7：保留。

#### DALI 105 固件更新透传命令

##### DATT_SEND32

在指定DALI通道所在的总线上发送32bit前向帧（Forward Frame) 命令，主要用于DALI 105固件更新功能的通信。

Data定义为：

| 1 byte  | 1 byte  | 1 byte  | 1 byte  | 1 byte  | 1 byte  |
| ------- | ------- | ------- | ------- | ------- | ------- |
| Channel | Control | Address | Opcode1 | Opcode2 | Opcode3 |

其中：

* **Channel**：DALI 通道，1字节。0：所有通道（默认）；1~4：通道编号；其他：保留未用。
* **Control**： 控制标志， 1字节。
* **Address**：DALI 105 地址字节， 1字节。
* **Opcode1/2/3**：DALI 105 操作码， 3字节。

**控制标志 Control**

| bit 7           | bit 6    | bit 5    | bit 4    | bit 3      | bit [2:0] |
| --------------- | -------- | -------- | -------- | ---------- | --------- |
| WaitForResponse | 保留未用 | 保留未用 | 保留未用 | Send-Twice | Priority  |

数据位定义如下：

- bit 7：**WaitForResponse**，等待响应数据帧。0：不等待响应；1：等待响应数据。
- bit [6:4]：保留未用。
- bit 3：**Send-Twice**，两次发送。0： 单次发送；1：发送2次命令（通常用于DALI命令中的Send-Twice命令）
- bit [2:0]：**Priority**, 传输优先级，默认为0，仅用于支持multi-master的应用中。0： 标准；1~5： 对应DALI标准中的Priority 0~4。6~7：保留。

#### DALI 透传命令应用示例

##### 102控制装置示例

以下实例以DALI通道1为例，所有Channel为0x01。

系统参数配置：

* NV_DATT_MONITOR_ENABLE = true
* NV_DATT_NO_WAIT_REPLY = false

**主机发送广播开灯100%命令**

主机发送的AsyncRequest请求帧，其中Data内容如下：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x00 | 不启用任何标志位，传输优先级默认。                           |
| Address       | 0xFE | DALI 16 bit 命令帧Address byte=0xFE， 高7位0x7F表示广播寻址，其中最低位Selector bit=0表示DAPC（Direct Arc Power Control）命令。 |
| Opcode        | 0xFE | DALI 16 bit 命令帧Opcodebyte=0xFE， 表示light output level为100%（参考DALI 102标准中的调光曲线说明）。 |
| AdditionData0 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData1 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData2 | 0x00 | 不适用，默认为0x00。                                         |

 模块依次返回：

1. SyncResponse：AckNack（默认应答帧），表示已正确接收并开始执行命令；
2. AsyncReport：DATT_DATA_IND（已发送16bit数据指示帧），汇报发送状态。

3. AsyncReport：DATT_DATA_IND，指示无需DALI总线响应数据。



**主机查询地址0x01的控制装置状态**

主机发送 AsyncRequest请求帧，其中Data内容如下：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x00 | 不启用任何标志位，传输优先级默认。                           |
| Address       | 0x03 | DALI 16 bit 命令帧Address byte=0x03， 高7位0x01表示单播地址为1，最低位Selector bit=1表示非DAPC命令。 |
| Opcode        | 0x90 | DALI 16 bit 命令帧Opcodebyte=0x90， 表示DALI 102命令 “QUERY STATUS”。 |
| AdditionData0 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData1 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData2 | 0x00 | 不适用，默认为0x00。                                         |

 模块依次返回：

1. SyncResponse：AckNack，表示已正确接收并开始执行命令；
2. AsyncReport：DATT_DATA_IND，汇报发送状态。
3. AsyncReport：DATT_DATA_IND，指示总线上接收到的响应帧（正常情况下为8 bit 后向帧数据）或DALI总线在标准允许的时间内未接收到响应帧。



**主机设置地址0x01的控制装置的PowerOnLevel参数值为254**

主机发送 AsyncRequest请求帧，为了提高效率，启用SetDTR0，数据帧的Data内容如下：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x18 | 0b00011000。SetDTR0=1，指示自动设置DTR0；Send-Twice=1，指示该命令为双次发送的配置命令；传输优先级默认。 |
| Address       | 0x03 | DALI 16 bit 命令帧Address byte=0x03， 高7位0x01表示单播地址为1，最低位Selector bit=1表示非DAPC命令。 |
| Opcode        | 0x2D | DALI 16 bit 命令帧Opcodebyte=0x2D， 表示DALI 102命令 “SET POWER ON LEVEL (DTR0)”。 |
| AdditionData0 | 0xFE | 表示自动设置的DTR0值为254。                                  |
| AdditionData1 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData2 | 0x00 | 不适用，默认为0x00。                                         |

 模块依次返回：

1. SyncResponse：AckNack，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_DATA_IND，汇报已发送“DTR0”（AdditionData0）命令。
3. AsyncReport：DATT_DATA_IND，汇报已发送第1次“SET POWER ON LEVEL (DTR0)”命令。
4. AsyncReport：DATT_DATA_IND，汇报已发送第2次“SET POWER ON LEVEL (DTR0)”命令。
5. AsyncReport：DATT_DATA_IND，指示无需总线响应。



**主机向地址为0x01的控制装置MemoryBank 136的位置16写入数据32**

DALI 102中向控制装置的指定MemoryBank的位置写入数据时需要分为下面几步：

1. 向地址0x01的控制装置发送 “ENABLE WRITE MEMORY”使能MemoryBank写，这样只有使能的控制装置会执行接下来的写入命令；
2. 发送“DTR1(136)”命令设置Memory Bank的序号位136；
3. 发送“DTR0(16)”命令设置Memory Bank内要写入的Location为16；
4. 发送“WRITE MEMORY LOCATION(DTR1,DTR0, 32) ”向Memory Bank的指定位置写入数据32。

> 注意：DALI 102的特殊命令如“DTR0”、“DTR1”和“WRITE MEMORY LOCATION”均为广播命令，无法使用单播或组播等其他寻址方式。因此向特定地址的控制装置的MemoryBank写数据，必须通过向特定地址的控制装置发送 “ENABLE WRITE MEMORY”命令使能MemoryBank写入来选中该控制装置，这是一种变通的寻址方式。在接收到和MemoryBank无关的任意其他命令后，控制装置会自动关闭MemoryBank写入使能。

将以上步骤转化为主机和模块之间的通信过程如下：

（1） 主机发送 AsyncRequest请求帧，向DALI总线发送“ENABLE WRITE MEMORY”，为了提高效率，启用SetDTR0将在后面要单独发送的“DTR0(16)”提前设置好，此处顺序调整不影响最终的执行目标。数据帧的Data内容如下：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x18 | 0b00011000。SetDTR0=1，指示自动设置DTR0；Send-Twice=1，指示该命令为双次发送的配置命令；传输优先级默认。 |
| Address       | 0x03 | DALI 16 bit 命令帧Address byte=0x03， 高7位0x01表示单播地址为1，最低位Selector bit=1表示非DAPC命令。 |
| Opcode        | 0x81 | DALI 16 bit 命令帧Opcodebyte=0x81， 表示DALI 102命令 “ENABLE WRITE MEMORY”。 |
| AdditionData0 | 0x10 | 表示自动设置的DTR0值为16。                                   |
| AdditionData1 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData2 | 0x00 | 不适用，默认为0x00。                                         |

 模块依次返回：

1. SyncResponse：AckNack，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_DATA_IND，汇报已发送“DTR0”（AdditionData0）命令。
3. AsyncReport：DATT_DATA_IND，汇报已发送第1次“ENABLE WRITE MEMORY”命令。
4. AsyncReport：DATT_DATA_IND，汇报已发送第2次“ENABLE WRITE MEMORY”命令。
5. AsyncReport：DATT_DATA_IND，指示无需总线响应。

（2）主机发送 AsyncRequest请求帧，向DALI总线发送“DTR1”命令设置MemoryBank序号为136，其中Data内容为：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x00 | 无使能标志位；传输优先级默认。                               |
| Address       | 0xC3 | DALI 16 bit 命令帧Address byte=0xC3， 表示DALI 102 特殊命令“DTR1(data)”。 |
| Opcode        | 0x88 | DALI 16 bit 命令帧Opcodebyte=0x88， 表示DTR1命令携带的数据data位136。 |
| AdditionData0 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData1 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData2 | 0x00 | 不适用，默认为0x00。                                         |

 模块依次返回：

1. SyncResponse：AckNack，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_DATA_IND，汇报已发送“DTR1（data）”命令。
3. AsyncReport：DATT_DATA_IND，指示无需总线响应。

（3）主机发送 AsyncRequest请求帧，“WRITE MEMORY LOCATION(DTR1,DTR0, 55) ”向Memory Bank的指定位置写入数据32，其中Data内容为：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x00 | 无使能标志位；传输优先级默认。                               |
| Address       | 0xC7 | DALI 16 bit 命令帧Address byte=0xC7， 表示DALI 102 特殊命令“WRITE MEMORY LOCATION(DTR1,DTR0, data)”。 |
| Opcode        | 0x20 | DALI 16 bit 命令帧Opcodebyte=0x20， 表示“WRITE MEMORY LOCATION”命令携带的数据data为32。 |
| AdditionData0 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData1 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData2 | 0x00 | 不适用，默认为0x00。                                         |

 模块依次返回：

1. SyncResponse：AckNack，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_DATA_IND，汇报已发送“WRITE MEMORY LOCATION(DTR1,DTR0, data)”命令。
3. AsyncReport：DATT_DATA_IND，指示无需总线响应。



**主机从地址为0x01的控制装置MemoryBank 136的位置16读出数据**

DALI 102标准中读Memory Bank数据分为以下几个步骤：

1. 发送“DTR1(136)”命令设置Memory Bank的序号位136；
2. 发送“DTR0(16)”命令设置Memory Bank内要写入的Location为16；
3. 发送“READ MEMORY LOCATION(DTR1,DTR0) ”从Memory Bank的指定位置读出数据。

将以上步骤转化为主机和模块之间的通信过程如下：

主机发送 AsyncRequest请求帧，向DALI总线发送“READ WRITE MEMORY”，为了提高效率，启用SetDTR0、SetDTR1设置好DTR0和DTR1，DTR0和DTR1的发送顺序不影响最终的执行目标。数据帧的Data内容如下：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x30 | 0b00110000。SetDTR1=1，指示自动设置DTR1；SetDTR0=1，指示自动设置DTR0；传输优先级默认。 |
| Address       | 0x03 | DALI 16 bit 命令帧Address byte=0x03， 高7位0x01表示单播地址为1，最低位Selector bit=1表示非DAPC命令。 |
| Opcode        | 0xC5 | DALI 16 bit 命令帧Opcodebyte=0xC5， 表示DALI 102命令 “READ MEMORY LOCATION(DTR1,DTR0) ”。 |
| AdditionData0 | 0x10 | 表示自动设置的DTR0值为16。                                   |
| AdditionData1 | 0x88 | 表示自动设置的DTR0值为136。                                  |
| AdditionData2 | 0x00 | 不适用，默认为0x00。                                         |

 模块依次返回：

1. SyncResponse：AckNack，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_DATA_IND，汇报已发送“DTR1”（AdditionData1）命令。
3. AsyncReport：DATT_DATA_IND，汇报已发送“DTR0”（AdditionData0）命令。
4. AsyncReport：DATT_DATA_IND，汇报已发送“READ MEMORY LOCATION(DTR1,DTR0) ”命令。
5. AsyncReport：DATT_DATA_IND，指示总线上接收到的响应帧（正常情况下为8 bit 后向帧数据）或DALI总线在标准允许的时间内未接收到响应帧。



**主机从地址为0x01的控制装置设置DT6扩展参数FastFadeTime的值为4**

按照DALI 102和207标准，对支持DT6的控制装置设置扩展参数FastFadeTime分为以下几个步骤：

1. 发送“DTR0(4)”命令设置将要执行的配置命令的数据为4；
2. 发送“ENABLE DEVICE TYPE 6”命令使能控制装置的扩展设备类型为6；
3. 发送“STORE DTR AS FAST FADE TIME”将控制装置FastFadeTime参数设置为DTR0中存放的数值4。

将以上步骤转化为主机和模块之间的通信过程如下：

主机发送 AsyncRequest请求帧，向DALI总线发送“READ WRITE MEMORY”，为了提高效率，启用SetDTR0、SetDTR1设置好DTR0和DTR1，DTR0和DTR1的发送顺序不影响最终的执行目标。数据帧的Data内容如下：

| 字段          | 数值 | 说明                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| Channel       | 0x01 | 通道1。                                                      |
| Control       | 0x98 | 0b10011000。SetDeviceType=1，指示自动设置DeviceType；SetDTR0=1，指示自动设置DTR0；Send-Twice=1，指示为双次发送配置命令；传输优先级默认。 |
| Address       | 0x03 | DALI 16 bit 命令帧Address byte=0x03， 高7位0x01表示单播地址为1，最低位Selector bit=1表示非DAPC命令。 |
| Opcode        | 0xE4 | DALI 16 bit 命令帧Opcodebyte=0xC5， 表示DALI 207扩展命令 “STORE DTR AS FAST FADE TIME”。 |
| AdditionData0 | 0x04 | 表示自动设置的DTR0值为4。                                    |
| AdditionData1 | 0x00 | 不适用，默认为0x00。                                         |
| AdditionData2 | 0x06 | 表示自动设置的DeviceType值为6。                              |

 模块依次返回：

1. SyncResponse：AckNack，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_DATA_IND，汇报已发送“DTR0”（AdditionData0）命令。
3. AsyncReport：DATT_DATA_IND，汇报已发送“ENABLE DEVICE TYPE X”（AdditionData2）命令。
4. AsyncReport：DATT_DATA_IND，汇报已发送第1次“STORE DTR AS FAST FADE TIME”命令。
5. AsyncReport：DATT_DATA_IND，汇报已发送第2次“STORE DTR AS FAST FADE TIME”命令。
6. AsyncReport：DATT_DATA_IND，指示无需总线响应。

##### 103控制设备示例

待添加。

### DALI 应用命令接口

DALI 应用命令接口（DALI Application Command Interface）用来对底层DALI数据传输进行封装，为上层业务提供简单的调用接口。使用该接口可以减少主机和模块的交互次数从而降低主机对DALI底层传输指令的控制需求，也可以提高通信效率改善用户体验，当然和透传接口相比，无法做到对DALI总线数据传输细节的监控。这些接口命令包括：
#### DALI 控制装置应用接口

##### DALI 控制装置搜索

主机发送DAA_CG_DISC命令请求模块开始搜索DALI总线上的102控制装置，模块执行搜索并返回DAA_CG_DISC_IND数据帧包括：

* 总线是否有设备在线；
* 总线上是否有未分配地址的设备；
* 总线上已分配地址的设备短地址（Short Address）列表。

指示搜索DALI总线过程中的结果，可在不同的阶段汇报多次结果。

###### DAA_CG_DISC

DAA_CG_DISC 命令PDU的data部分内容为：

| 1 byte  |      1 byte      |   1 byte   |
| :-----: | :--------------: | :--------: |
| Channel | DiscoveryOptions | TargetAddr |

其中，

* Channel：DALI 通道编号。
* DiscoveryOptions：搜索选项。
* TargetAddr：搜索目标地址。0~63， 短地址指定的单个设备；65~79，对应组地址0~15，单个组地址；126，对应未分配地址的所有设备；127，对应所有设备，此时会执行短地址空间扫描。

**DiscoveryOptions**

| bit 7:5  |  bit 4   |    bit 3     |  bit 2   | bit 1:0 |
| :------: | :------: | :----------: | :------: | :-----: |
| 保留未用 | 保留未用 | ForceRestart | NoIntInd |  Mode   |

其中，

* ForceRestart：如果有未完成的搜索，是否强制重启搜索。0：否，等待当前搜索完成；1：是，强制结束当前未完成的搜索，重新启动搜索。
* NoIntInd：No Intermediate Indication，不需要搜索过程中汇报进度。0：搜索过程中汇报状态变化；1：不汇报，直到搜索完成后才汇报搜索结果。
* Mode：搜索模式，取值0~3。注意搜索模式和搜索目标地址是不同的，使用二者的组合可以实现多种搜索意图。
  * 0：搜索返回地址存在设备、存在重复设备和不存在设备的指示；
  * 1：搜索仅返回地址存在设备的指示；
  * 2：搜索仅返回地址存在重复设备的指示；
  * 3：搜索仅返回地址不存在设备的指示。

###### DAA_CG_DISC_IND

DAA_CG_DISC_IND 数据帧用于汇报设备搜索结果，搜索过程中每当发现新的设备，模块都会发送DAA_CG_DISC_IND 用于汇报状态。

DAA_CG_DISC_IND 数据帧的PDU data定义如下：

| 1 byte  |  1 byte   |   1 byte   |  1 byte/16 bytes   |
| :-----: | :-------: | :--------: | :----------------: |
| Channel | DiscState | TargetAddr | Present/PresentMap |

其中，

* Channel：DALI 通道编号。
* DiscState：搜索状态，取值0~15。0： Running，进行中未完成；1：Finished，正常结束；2：Pending， 设备或者总线忙，等待；3：Exception， 异常退出； 4~15：保留 。
* TargetAddr：当前搜索的目标设地址。0~63：短地址；126：未分配地址的所有设备，仅当没有搜索到设备时使用；127：所有设备，仅当DiscState为Finished时使用。
* Present：当前搜索设备短地址是否存在的结果。仅当DiscState为Running时，汇报单个短地址搜索结果。0： NO，不存在；1：YES，存在单个设备；2：存在多个设备（地址重复）；3：未知错误。4~255：保留未用。
* PresentMap：总线地址搜索结果分布。仅当DiscState为Finished时，汇报整个地址空间搜索的结果。
  * 16 bytes，共128 bits。 
  * 高位在前，每2个bit表示一个ShortAddress扫描的结果。
  * 2-bit的状态值取值为0~3。0：不存在；1：存在；2：存在重复地址设备；3：未知。


##### DALI 控制装置地址分配

###### DAA_CG_ADDRESSING

DAA_CG_ADDRESSING 命令PDU的data部分内容为：

| 1 byte  |      1 byte       |
| :-----: | :---------------: |
| Channel | AddressingOptions |



**AddressingOptions**

| bit 7     | bit 6        | bit 5 | bit 4         | bit 3   | bit 2        | bit 1    | bit 0      |
| --------- | ------------ | ----- | ------------- | ------- | ------------ | -------- | ---------- |
| QueryInfo | RemoveGroups | VisFb | IgnoreNotRspd | QueryDT | ForceRestart | NoIntInd | UnaddrOnly |

其中，

* QueryInfo：查询设备的基本信息，如
    * DALI Version Number
    * GTIN 0~5
    * Device Type List
    * 102 Control gear 参数
        * STATUS
        * ACTUAL LEVEL
        * MIN LEVEL
        * MAX LEVEL
        * FADE TIME/FADE RATE
        * PHYSICAL MINIMUM LEVEL
        * GROUPS 0-7, 8-15


* RemoveGroups：移除分组。
* VisFb：Visible Feedback分配过程是否需要视觉反馈。0：无反馈；1：分配开始先关闭待分配的设备，在分配过程中依次点亮获得地址的设备。
* IgnoreNotRspd: Ignore Not Responding，忽略不响应的设备。0：如果有不响应的设备则停止分配；1：忽略不响应的设备。 
* QueryDT：Query Device Type，需要查询设备类型。0：不需要，1：需要，此处查询仅返回单个DeviceType类型或者指示存在多个设备类型，多个设备类型仍需要主机发起独立查询请求来获取控制装置支持的所有设备类型列表。
* ForceRestart：如果有未完成的分配，是否强制重启分配。0：否，等待当前分配完成；1：是，强制结束当前未完成的分配，重新启动分配。
* NoIntInd：No Intermediate Indication，不需要分配过程中汇报进度。0：分配过程中汇报状态变化；1：不汇报，直到分配完成后才汇报分配结果。
* UnaddrOnly：Unaddressed Only,只对未分配地址执行地址分配。0：已有地址的设备将会先被删除地址，然后对所有设备执行地址重新分配，适用于全新安装；1：仅对未分配地址的设备执行地址分配，用于系统扩展。

###### DAA_CG_ADDRESSING_WITH_MASK

DAA_CG_ADDESSING_WITH_MASK命令PDU的data部分内容为：

| 1 byte  |      1 byte       |     8 bytes     |
| :-----: | :---------------: | :-------------: |
| Channel | AddressingOptions | AllowedAddrMask |

其中，

* Channel：DALI 通道编号。
* AddressingOptions：地址分配选项。参考DAA_CG_ADDESSING的AddressingOptions说明。
* AllowedAddrMask: 8 字节，高字节在前，每个bit对应63~0共64个短地址允许位。每个字节均为0xFF表示每个地址都是允许的。

###### DAA_CG_ADDRESSING_IND

DAA_CG_ADDRESSING_IND数据帧用于汇报设备地址分配结果，搜索过程中每当对设备进行地址分配，模块都会发送DAA_CG_ADDRESSING_IND用于汇报状态。

DAA_CG_ADDRESSING_IND数据帧的PDU data定义如下：

| 1 byte  |     1 byte      | 1 byte |      1 byte      |
| :-----: | :-------------: | :----: | :--------------: |
| Channel | AddressingState |  Addr  | AddressingResult |

其中，

* Channel：DALI 通道编号。
* AddressingState：地址分配状态，取值0~15。0： Running, 进行中未完成；1：Finished, 正常结束；2：Pending， 设备或者总线忙，等待；3：Exception； 6~15：保留 。
* Addr：
  * AddressingState=Running时，表示当前尝试分配的设备短地址（0~63）。
  * AddressingState=Finished时，固定为0xFF。
  * AddressingState为其他值时，0~63表示结束时正在分配的地址，0xFF表示无可用合法地址。

* AddressingResult：
  * AddressingState=Running时，表示对设备短地址分配（编程）的结果。0： 失败，未完成分配；255：成功，对单个设备完成地址分配；254：分配失败，存在多个设备（地址重复）；253：未知错误； 252：DeviceNotResponding，设备不响应；251：总线上设备超过64个；250：没有可供分配的短地址；249：Timeout，超时；1~252：保留未用。
  * AddressingState为其他值时，表示新增完成地址分配的设备数量，0~64。

##### DALI 控制装置控制、配置和查询

DALI总线每次只能传输2个字节的前向帧，设备返回1个字节的后向帧作为应答，效率很低。主机若参与每次传输的控制流程，也较为繁琐。因此模块也为典型应用提供了一些较为高级的宏命令（Macro）接口，这些接口可能涉及到DALI总线山多个传输过程，但是主机只需要监测模块返回的最终结果的确认或者响应命令，无需了解细节。

如果这里的应用接口不能满足主机的需求，主机仍然可以使用 [DALI 透传接口](#DALI 透传接口)完成所要的功能。

###### DAA_CG_DAPC

直接电弧功率控制。Data 部分格式：

| 1 byte  | 1 byte  | 1 byte |
| :-----: | :-----: | :----: |
| Channel | Address | Level  |

其中，

* Channel：DALI 通道。
* Address：目标控制装置地址。0~63：短地址；64~79：（64+组地址）；126：广播（未分配地址设备）；127：广播（所有设备）。
* Level：功率等级。0：关闭；254：100%；255：不改变，用于某些特殊场合。



先返回 AckNack，在命令执行后返回 DAA_CG_DAPC_CFM。

**DAA_CG_DAPC_CFM**

| 1 byte  | 1 byte  | 1 byte | 1 byte |
| :-----: | :-----: | :----: | :----: |
| Channel | Address | Level  | Status |

其中，

* Channel：和请求帧中的字段含义相同。
* Address：和请求帧中的字段含义相同。
* Level：和请求帧中的字段含义相同。
* Status：命令执行结果状态。具体含义参考 [DALI宏命令执行结果状态](#DALI宏命令执行结果状态)。



**DALI宏命令执行结果状态**

| 名称                              | 数值 | 含义                                                       |
| --------------------------------- | ---- | ---------------------------------------------------------- |
| DALI_MACRO_RESULT_OK              | 0    | 宏命令执行成功                                             |
| DALI_MACRO_RESULT_BUS_ERR         | 1    | 总线故障                                                   |
| DALI_MACRO_RESULT_MODE_INIT       | 2    | DALI在初始化过程中                                         |
| DALI_MACRO_RESULT_MODE_QUIESCENT  | 3    | DALI处在静默模式（多主机系统接受到其他主机的请求静默命令） |
| DALI_MACRO_RESULT_TX_BUF_FULL     | 4    | 发送缓冲区满                                               |
| DALI_MACRO_RESULT_CHANNEL_NA      | 5    | DALI通道不可用                                             |
| DALI_MACRO_RESULT_INVALID_PARA    | 6    | DALI宏命令携带非法参数                                     |
| DALI_MACRO_RESULT_BUSY_RUNNING    | 7    | 当前有其他宏命令在运行                                     |
| DALI_MACRO_RESULT_NO_RSPD         | 8    | 目标设备没有响应                                           |
| DALI_MACRO_RESULT_TRASHED         | 9    | 目标设备响应但数据帧损坏（通常为多个设备响应）             |
| DALI_MACRO_RESULT_TERMINATED      | 10   | 主机发送宏命令终止请求后停止执行                           |
| DALI_MACRO_RESULT_UNKNOWN_FAILURE | 255  | 未知故障                                                   |

###### DAA_CG_CTRL

发送DALI 102 控制命令，控制命令无需DALI总线设备响应。包括以下命令：

| 命令名称  | OpCode  | 描述 |
| :-----: | :-----: | :----: |
| OFF | 0x00 | 关闭 |
| UP | 0x01 |  |
| DOWN | 0x02 |  |
| STEP UP | 0x03 |  |
| STEP DOWN | 0x04 |  |
| RECALL MAX LEVEL | 0x05 |  |
| RECALL MIN LEVEL | 0x06 |  |
| STEP DOWN AND OFF | 0x07 |  |
| ON AND STEP UP | 0x08 |  |
| ENABLE DAPC SEQUENCE | 0x09 |  |
| GO TO LAST ACTIVE LEVEL | 0x0A |  |
| GO TO SCENE (sceneNumber) | 0x10+sceneNumber |  |


命令Data格式：

| 1 byte  | 1 byte  | 1 byte |
| :-----: | :-----: | :----: |
| Channel | Address | OpCode |

其中，

* Channel：DALI 通道。
* Address：目标控制装置地址。0~63：短地址；64~79：（64+组地址）；126：广播（未分配地址设备）；127：广播（所有设备）。
* OpCode：控制装置控制指令码。参考上述列表。

先返回 AckNack，在命令执行后返回 DAA_CG_CTRL_CFM。

**DAA_CG_CTRL_CFM**

| 1 byte  | 1 byte  | 1 byte | 1 byte |
| :-----: | :-----: | :----: | :----: |
| Channel | Address | OpCode | Status |

其中，

* Channel：和请求帧中的字段含义相同。
* Address：和请求帧中的字段含义相同。
* OpCode：和请求帧中的字段含义相同。
* Status：命令执行结果状态。具体含义参考 [DALI宏命令执行结果状态](#DALI宏命令执行结果状态)。

###### DAA_CG_CFG

发送DALI 102 配置命令，配置命令一般带有参数，无需DALI总线设备回复，一般需要连续发送两次 （Send-twice)。包括以下命令：

|             命令名称              |      OpCode      | 参数 | 描述 |
| :-------------------------------: | :--------------: | ---- | :--: |
|               RESET               |       0x20       |      | 复位 |
|    STORE ACTUAL LEVEL IN DTR0     |       0x21       | DTR0 |      |
|     SAVE PERSISTENT VARIABLES     |       0x22       |      |      |
|    SET OPERATING MODE (*DTR0*)    |       0x23       | DTR0 |      |
|    RESET MEMORY BANK (*DTR0*)     |       0x24       | DTR0 |      |
|          IDENTIFY DEVICE          |       0x25       |      |      |
|      SET MAX LEVEL (*DTR0*)       |       0x2A       | DTR0 |      |
|      SET MIN LEVEL (*DTR0*)       |       0x2B       | DTR0 |      |
| SET SYSTEM FAILURE LEVEL (*DTR0*) |       0x2C       | DTR0 |      |
|   SET POWER ON LEVEL  (*DTR0*)    |       0x2D       | DTR0 |      |
|      SET FADE TIME  (*DTR0*)      |       0x2E       | DTR0 |      |
|      SET FADE RATE  (*DTR0*)      |       0x2F       | DTR0 |      |
|  SET EXTENDED FADE TIME (*DTR0*)  |       0x30       | DTR0 |      |
|    SET SCENE (*DTR0, sceneX*)     | 0x40+sceneNumber | DTR0 |      |
|   REMOVE FROM SCENE (*sceneX*)    | 0x50+sceneNumber |      |      |
|      ADD TO GROUP (*group*)       |    0x60+group    | DTR0 |      |
|    REMOVE FROM GROUP (*group*)    |    0x70+group    |      |      |
|    SET SHORT ADDRESS (*DTR0*)     |       0x80       | DTR0 |      |
|        ENABLE WRITE MEMORY        |       0x81       |      |      |


命令Data格式：

| 1 byte  | 1 byte  | 1 byte | 1 byte  | 1 byte |
| :-----: | :-----: | :----: | :-----: | :----: |
| Channel | Address | OpCode | Options | Value  |

其中，

* Channel：通道。
* Address：DALI 设备地址。
* OpCode：DALI 102 控制装置命令，参考上面所列的配置命令列表。
* Options：选项字节。
  * bit[1:0]： DTR0Mode，指示是否自动设置DTR0为Value数值。0：Auto，自动由模块根据标准中配置命令的DTR0需求决定；1：ForceSet，强制自动设置DTR0；2：ForceNotSet，强制不自动设置DTR0；3：保留未用。
  * bit[7:2]：保留未用。
* Value：数值。当需要自动设置数值时，模块将其作为DTR0的值发送出去。

先返回 AckNack，在命令执行后返回确认命令 DAA_CG_CFG_CFM。

**DAA_CG_CFG_CFM**

| 1 byte  | 1 byte  | 1 byte | 1 byte |
| :-----: | :-----: | :----: | :----: |
| Channel | Address | OpCode | Status |

其中，

* Channel：和请求帧中的字段含义相同。
* Address：和请求帧中的字段含义相同。
* OpCode：和请求帧中的字段含义相同。
* Status：状态。具体含义参考 [DALI宏命令执行结果状态](#DALI宏命令执行结果状态)。

###### DAA_CG_QUERY

发送DALI 102 查询命令，查询命令可以无需参数或者带有参数，需要DALI总线设备回复。包括以下命令：

|              命令名称               |      OpCode      | 参数     | 描述 |
| :---------------------------------: | :--------------: | -------- | :--: |
|            QUERY STATUS             |       0x90       |          |      |
|     QUERY CONTROL GEAR PRESENT      |       0x91       |          |      |
|         QUERY LAMP FAILURE          |       0x92       |          |      |
|         QUERY LAMP POWER ON         |       0x93       |          |      |
|          QUERY LIMIT ERROR          |       0x94       |          |      |
|          QUERY RESET STATE          |       0x95       |          |      |
|     QUERY MISSING SHORT ADDRESS     |       0x96       |          |      |
|        QUERY VERSION NUMBER         |       0x97       |          |      |
|         QUERY CONTENT DTR0          |       0x98       | DTR0     |      |
|          QUERY DEVICE TYPE          |       0x99       |          |      |
|       QUERY PHYSICAL MINIMUM        |       0x9A       |          |      |
|         QUERY POWER FAILURE         |       0x9B       |          |      |
|         QUERY CONTENT DTR1          |       0x9C       | DTR1     |      |
|         QUERY CONTENT DTR2          |       0x9D       | DTR2     |      |
|        QUERY OPERATING MODE         |       0x9E       |          |      |
|       QUERY LIGHT SOURCE TYPE       |       0x9F       | DTR0/1/2 |      |
|         QUERY ACTUAL LEVEL          |       0xA0       |          |      |
|           QUERY MAX LEVEL           |       0xA1       |          |      |
|           QUERY MIN LEVEL           |       0xA2       |          |      |
|        QUERY POWER ON LEVEL         |       0xA3       |          |      |
|     QUERY SYSTEM FAILURE LEVEL      |       0xA4       |          |      |
|      QUERY FADE TIME/FADE RATE      |       0xA5       |          |      |
|  QUERY MANUFACTURER SPECIFIC MODE   |       0xA6       |          |      |
|       QUERY NEXT DEVICE TYPE        |       0xA7       |          |      |
|      QUERY EXTENDED FADE TIME       |       0xA8       |          |      |
|     QUERY CONTROL GEAR FAILURE      |       0xAA       |          |      |
|    QUERY SCENE LEVEL (*sceneX*)     | 0xB0+sceneNumber |          |      |
|          QUERY GROUPS 0-7           |       0xC0       |          |      |
|          QUERY GROUPS 8-15          |       0xC1       |          |      |
|      QUERY RANDOM ADDRESS (H)       |       0xC2       |          |      |
|      QUERY RANDOM ADDRESS (M)       |       0xC3       |          |      |
|      QUERY RANDOM ADDRESS (L)       |       0xC4       |          |      |
| READ MEMORY LOCATION (*DTR1, DTR0*) |       0xC5       | DTR0/1   |      |
|    QUERY EXTENDED VERSION NUMBER    |       0xFF       |          |      |

命令Data格式：

| 1 byte  | 1 byte  | 1 byte |
| :-----: | :-----: | :----: |
| Channel | Address | OpCode |

其中，

* Channel：通道。
* Address：DALI 设备地址。
* OpCode：DALI 102 控制装置命令，参考上面所列的配置命令列表。

先返回AckNack，命令执行后再返回 DAA_CG_QUERY_RSP。

**DAA_CG_QUERY_RSP**

| 1 byte  | 1 byte  | 1 byte | 1 byte | 1 byte |
| :-----: | :-----: | :----: | :----: | :----: |
| Channel | Address | OpCode | Status | Answer |

其中，

* Channel：和请求帧中的字段含义相同。
* Address：和请求帧中的字段含义相同。
* OpCode：和请求帧中的字段含义相同。
* Status：状态。具体含义参考 [DALI宏命令执行结果状态](#DALI宏命令执行结果状态)。

###### DAA_CG_MB_READ

读取102 控制装置的MemoryBank。



命令Data格式：

| 1 byte  | 1 byte  | 1 byte  | 1 byte | 1 byte | 1 byte |
| :-----: | :-----: | :-----: | :----: | :----: | :----: |
| Channel | Address | Options |  Bank  | First  |  Last  |

先返回AckNack，命令执行后再返回 DAA_CG_MB_READ_RSP。



###### DAA_CG_MB_WRITE

###### DAA_CG_MB_ACCESS

DALI 控制装置Memory Bank操作。

命令Data格式：

| 1 byte  | 1 byte  | 1 byte  | 1 byte | 1 byte | 1 byte |
| :-----: | :-----: | :-----: | :----: | :----: | :----: |
| Channel | Address | Options |  Bank  | First  |  Last  |

先返回AckNack，命令执行后再返回 DAA_CG_MB_ACCESS_RSP。



###### DAA_CG_APP_CTRL

###### DAA_CG_APP_CFG

###### DAA_CG_APP_QUERY

#### DALI 控制设备应用接口

暂未定义。



## 主机侧完整应用示例

## 附录
### DALI 102控制装置命令列表
此列表依据IEC 62386-102 ed2.0。



### DALI 103控制设备命令列表

暂未使用。
