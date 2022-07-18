# MDA180 DALI AC 接口模块通信协议

版本：v0.4， 更新日期：2022-07-14

2022(c) 南京美加杰智能科技有限公司 www.meijay.com



本文档定义了MDA180 DALI **ACMI** （**A**pplication **C**ontroller  **M**odule **I**nterface，应用控制器模块接口 ）通信协议。

## UART 标准协议

### 硬件接口

硬件接口默认为通用异步串行口(UART)，3.3V TTL逻辑电平。

引脚包括：

* **MDA_TXD**：模块发送。
* **MDA_RXD**：模块接收。
* **MDA_NRST**： 模块MCU复位，低电平复位，高电平释放。

### UART参数设置

UART通信参数设置为：**115200bps**, **8**-**N**-**1**。 每个字节的传输时间约为86.8us。

### UART 传输时序
* 数据帧内字节传输间隔  < 1.5个字节传输长度 =~130us。
* 数据帧之间的静默时间  > 3.5个字节传输长度 =~300us。

## 数据帧格式

### 一般格式

**数据帧**的长度为 6~256 字节， 组成为：

| 1 byte |  1 byte  | 3~253 bytes | 1 byte |
| :----: | :------: | :---------: | :----: |
|  SOF   | SiteAddr |     PDU     |  FCS   |

其中：

- **SOH**:  帧起始标志，1字节，等于0xFE。（备注：如果使用了数据帧间的时序要求，起始字节似乎没有必要）。
- **SiteAddr**： 从站地址，1字节，取值范围0~255，使用UART接口默认为0；使用RS485接口时表示从站地址，0表示广播。
- **PDU**: PDU（**P**rotocol **D**ata **U**nit，协议数据单元）， 3~253字节，格式含义根据FrameType不同分别定义。
- **FCS**:  1 字节，SiteAddr和PDU部分内容按字节 XOR 校验和 。



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



### PDU格式

PDU的组成如下：

| 1 byte | 1 byte | 1 byte | 1 byte | 0~249 byte(s) |
| :----: | :----: | :----: | :----: | :-----------: |
| Length |  Seq   | Header | CmdId  |     Data      |

其中：

- **Length**：Data 长度， 1字节，取值范围 0~249。
- **Seq**：帧序列号，1字节。一般由主机指定，模块返回与之相关的响应帧应该使用相同的序列号以便主机追溯， 通常应采取循环递增的策略。模块主动发起的传输该位由模块自行指定，通常默认为0。
- **Header**：帧头， 1字节。
  - Bit[7]：**Direction**， 方向。0：主机发送请求；1：模块发送响应或报告；
  - Bit[6:4]：**FrameType**， 帧类型，取值0~7。
  - Bit[3:0]：**CmdCat**， 命令类别，取值0~15。具体取值含义参见DALI ACI段落。
- **CmdId**：命令标识， 1字节，取值范围0~255。
- **Data**: 有效数据，0~249字节。

#### 帧类型列表

FrameType指示数据帧的类型。

| FrameType | 名称             | 说明                                                         |
| --------- | ---------------- | ------------------------------------------------------------ |
| 0         | **Poll**         | 轮询获取队列消息（仅限于RS485和SPI接口），一般用于主机发送给模块 |
| 1         | **SyncRequest**  | 同步请求，期望获得SyncResponse，一般用于主机发送给模块       |
| 2         | **AsyncRequest** | 异步请求，主机发送给模块                                     |
| 3         | **SyncResponse** | 同步响应，接收到SyncRequest后立即发出的响应，一般用于模块发送给主机 |
| 4         | **AsyncReport**  | 异步汇报，一般用于模块发送给主机                             |
| 5         |                  | 保留未用                                                     |
| 7         | **Exception**    | 异常                                                         |

####子系统列表

DALI **ACMI** 数据帧命令类别（CmdCat）如下表所列：

| CmdCat |             Name              |  Description  |
| :----: | :---------------------------: | :-----------: |
|   0    |           Reserved            |     保留      |
|   1    |       System Management       |   系统管理    |
|   2    |    DALI Channel Management    | DALI 通道管理 |
|   3    | DALI Transparent Transmission |   DALI 透传   |
|   4    |   DALI Application Command    | DALI 应用命令 |
|  5~15  |           Reserved            |     保留      |

#### 默认应答帧（DEFAULT_RSP）

默认应答帧仅用于模块确认接收到主机的请求帧， 一般用于无需响应数据的请求帧。其PDU部分的内容定义如下：

| 字段           | 数值             | 说明                      |
| -------------- | ---------------- | ------------------------- |
| Length         | 0              |                           |
| Seq | 和主机请求帧相同 |  |
| Header         | 0b1011xxxx       | Direction=1，模块发送响应; FrameType=3，同步响应; CmdCat和主机请求帧中的数据相同。 |
| CmdId  | 和主机请求帧相同 |  |
| Data           |      | 无。 |

#### 异常帧（Exception Frame）

异常帧用于模块向主机指示接收到的数据帧中存在错误，无法执行。

异常代码存放在异常帧的Data中。

PDU定义如下：

| 字段   | 数值             | 说明                                                         |
| ------ | ---------------- | ------------------------------------------------------------ |
| Length | 1                |                                                              |
| Seq    | 和主机请求帧相同 |                                                              |
| Header | 0b1011xxxx       | Direction=1，模块发送响应; FrameType=7，异常帧; CmdCat和主机请求帧中的数据相同。 |
| CmdId  | 和主机请求帧相同 |                                                              |
| Data   | ErrorCode        | 数据长度为1字节，表示错误代码，含义详见“错误代码”列表。      |

**错误代码列表**

| ErrorCode | 名称                   | 含义                                                |
| --------- | ---------------------- | --------------------------------------------------- |
| 1         | Illegal Command        | 非法命令，CmdCat和CmdId组合不是模块支持的命令功能。 |
| 2         | Illegal Data           | 非法数据，数据帧携带的Data不是合法数据。            |
| 3         | Module Service Busy    | 模块服务忙，无法处理所请求的命令。                  |
| 4         | Module Service Failure | 模块服务失效，执行请求命令失败。                    |
| 5         |                        |                                                     |

## 通信流程

### 主机发送请求

一般情况下，通信均由主机发起，主机向模块发送请求，模块响应主机的请求。

模块响应可能包含多个响应数据帧，如下：

* 模块接收到主机请求后，立即回复默认应答帧（DEFAULT_RSP），此应答中Data部分只包含和主机请求帧中相同的CommandType和CmdId， 用于确认主机请求帧的正确接收；如模块判断接收帧功能非法或自身无法提供提供所需要的服务，则立即返回错误帧（Exception）。
* 若该请求为异步操作（如发送DALI透传命令或执行DALI应用命令），则模块在DALI发送完成后向主机发送汇报数据帧。
* 若该请求为包含多个步骤的异步操作（如DALI透传查询命令或DALI应用命令），则模块在异步操作的每一步状态变化时（例如收到DALI总线上的控制装置响应数据）都会向主机发送汇报数据帧。

多个响应之间的超时控制请参考每个数据帧的定义说明。

### 模块主动上报

模块在以下几种情况下会向主机主动发起上报：

* 执行耗时较长的异步操作过程中状态发生了变化，或操作完成时。
* 检测到DALI总线状态变化或检测到了数据。
* 定时任务。

> 注意： 若为RS485应用，应禁用模块主动上报功能，而由主机采用轮询的方式来读取模块上报的消息。


## 接口命令定义

### 系统管理接口

CmdCat为  0x01。

模块系统管理接口（System Management Interface）命令包括：

| Direction | FrameType    | 名称              | CmdId | 描述             | 响应                       |
| --------- | ------------ | ----------------- | ----- | ---------------- | -------------------------- |
| 0         | SyncRequest  | **SYS_RESET_REQ** | 0x00  | 请求模块复位     | DEFAULT_RSP，SYS_RESET_IND |
| 1         | AsyncReport  | **SYS_RESET_IND** | 0x01  | 模块复位指示     | 无                         |
| 0         | SyncRequest  | **SYS_VERSION**   | 0x02  | 获取模块版本信息 | SYS_VERSION_RSP            |
| 0         | SyncRequest  | **SYS_READ_CFG**  | 0x03  | 读取模块配置数据 | SYS_READ_CFG_RSP           |
| 0         | SyncRequest  | **SYS_WRITE_CFG** | 0x04  | 写入模块配置数据 | SYS_WRITE_CFG_RSP          |
| 0         | SyncRequest  |                   | 0x05  |                  |                            |
| 1         | SyncResponse | SYS_VERSION_RSP   | 0x02  |                  |                            |

#### 系统复位



#### 系统版本和信息查询

#### 系统配置参数

配置参数分为掉电不丢失（Non-Volatile）和掉电丢失（Volatile）两种，分别用NV和VOL开头表示。

NV配置数据在模块重启复位后仍然有效，VOL配置数据需要每次复位后重新写入。

##### SYS_READ_CFG

读取配置参数。

格式：CfgId 



回复： SYS_READ_CFG_RSP

Status | CfgId | CfgLength | CfgValue 

##### SYS_WRITE_CFG

写入配置参数。

格式：CfgId | CfgLength | CfgValue 



回复：SYS_WRITE_CFG_RSP

格式：Status



**系统参数列表**

| ParameterId（2 bytes） | DataType（1 byte） | 名称                   | 说明                                                         |
| ---------------------- | ------------------ | ---------------------- | ------------------------------------------------------------ |
| 0x0010                 | BOOL               | NV_DATT_MONITOR_ENABLE | DALI透传监控使能，开启后模块将返回所有接收到的总线数据，包括异常数据，一般用于总线监控分析。 |
| 0x0011                 | BOOL               | NV_DATT_ECHO_ENABLE    | DALI透传自发数据帧回复使能，开启后每次透传发送的数据也会返回，否则只返回命令已发送的确认帧。默认为false, |
| 0x0012                 | BOOL               | NV_DATT_WAITREPLY      | DALI透传等待响应，开启后每次透传均等待总线从机响应直到后向帧超时。 |
|                        |                    | NV_DATT_NO_ECHO        |                                                              |
|                        |                    |                        |                                                              |



### DALI 通道管理接口
CmdCat  为  0x02。
DALI 通道管理接口（DALI Channel Management Interface）命令包括：

| Direction | FrameType    | 名称                        | CmdId | 描述                                 | 响应                    |
| --------- | ------------ | --------------------------- | ----- | ------------------------------------ | ----------------------- |
| 0         | SyncRequest  | **DACM_INFO**               | 0x00  | 获取DALI信息                         | DACM_INFO_RSP           |
| 0         | SyncRequest  | **DACM_START**              | 0x10  | 开启DALI通道                         | DEFAULT_RSP             |
| 0         | SyncRequest  | **DACM_STOP**               | 0x11  | 关闭DALI通道                         | DEFAULT_RSP             |
| 0         | SyncRequest  | **DACM_STATUS**             | 0x12  | 获取DALI通道状态                     | DACM_STATUS_RSP         |
| 0         | SyncRequest  | **DACM_CONTACT_ON**         | 0x13  | 打开DALI通道对应的回路干接点信号     | DEFAULT_RSP             |
| 0         | SyncRequest  | **DACM_CONTACT_OFF**        | 0x14  | 关闭DALI通道对应的回路干接点信号     | DEFAULT_RSP             |
| 0         | SyncRequest  | **DACM_CONTACT_STATUS**     | 0x15  | 获取DALI通道对应的回路干接点信号状态 | DACM_CONTACT_STATUS_RSP |
| 1         | SyncResponse | **DACM_INFO_RSP**           | 0x00  | DALI信息响应                         | 无                      |
| 1         | SyncResponse | **DACM_STATUS_RSP**         | 0x12  | DALI通道状态响应                     | 无                      |
| 1         | SyncResponse | **DACM_CONTACT_STATUS_RSP** | 0x15  | DALI通道对应的回路干接点信号状态响应 | 无                      |

#### DALI 通道基本管理

##### DACM_INFO

获取DALI信息，包括通道数、是否集成BPS、BPS最大电流等。

Data内容为空。

##### DACM_INFO_RSP

响应帧的Data内容定义如下：

| 1 byte           | 1 byte        | 1 byte        |
| ---------------- | ------------- | ------------- |
| NumberOfChannels | BPSIntegrated | BPSMaxCurrent |

#### DALI 通道开关控制

##### DACM_START

开启DALI通道功能，模块将开启数据收发功能。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |



回复DEFAULT_RSP。

##### DACM_STOP

停止DALI通道功能，模块将关闭数据收发功能。

Data 内容定义如下：

| 1 byte  |
| ------- |
| Channel |



回复DEFAULT_RSP。

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

#### DALI 通道回路干接点控制

##### DACM_CONTACT_ON

##### DACM_CONTACT_OFF

##### DACM_CONTACT_STATUS

##### DACM_CONTACT_STATUS_RSP

### DALI 透传接口
CmdCat为  0x03。

DALI 透传接口适用于主机自行实现DALI应用层管理，支持在模块DALI通道总线上收发DALI数据帧。

DALI 透传接口（DALI Transparent Transmission Interface）命令包括：

| Direction | FrameType    | 名称                | CmdId | 描述                | 响应                                        |
| --------- | ------------ | ------------------- | ----- | ------------------- | ------------------------------------------- |
| 0         | AsyncRequest | **DATT_SEND**       | 0x00  | 发送DALI总线数据    | DEFAULT_RSP，DATT_SEND_CFM，DATT_RECV_IND   |
| 0         | AsyncRequest | **DATT_SEND8**      | 0x01  | 发送8bit数据        | DEFAULT_RSP，DATT_SEND8_CFM，DATT_RECV_IND  |
| 0         | AsyncRequest | **DATT_SEND16**     | 0x02  | 发送16bit数据       | DEFAULT_RSP，DATT_SEND16_CFM，DATT_RECV_IND |
| 0         | AsyncRequest | **DATT_SEND24**     | 0x03  | 发送24bit数据       | DEFAULT_RSP，DATT_SEND24_CFM，DATT_RECV_IND |
| 0         | AsyncRequest | **DATT_SEND32**     | 0x04  | 发送32bit数据       | DEFAULT_RSP，DATT_SEND32_CFM，DATT_RECV_IND |
| 0         | Poll         | **DATT_RECV_POLL**  | 0x10  | 查询接收数据        | DATT_RECV_IND                               |
| 1         | AsyncReport  | **DATT_SEND_CFM**   | 0x00  | 已发送DALI数据确认  | 无 可能和DATT_RECV_IND是一个意思？          |
| 1         | AsyncReport  | **DATT_SEND8_CFM**  | 0x01  | 已发送8bit数据确认  | 无                                          |
| 1         | AsyncReport  | **DATT_SEND16_CFM** | 0x02  | 已发送16bit数据确认 | 无                                          |
| 1         | AsyncReport  | **DATT_SEND24_CFM** | 0x03  | 已发送24bit数据确认 | 无                                          |
| 1         | AsyncReport  | **DATT_SEND32_CFM** | 0x04  | 已发送32bit数据确认 | 无                                          |
| 1         | AsyncReport  | **DATT_RECV_IND**   | 0x10  | 接收数据指示        | 无                                          |

透传命令的使用流程如下：

* 主机发送DATT_SENDxxx命令；
* 模块接收后回复DEFAULT_RSP确认；
* 模块在DALI数据传输完成后汇报DATT_SENDxxx_IND告知主机发送情况；
* 模块最后向主机发送DATT_RECV_IND指示总线接收情况。

> **注意**：可以通过发送模块系统配置命令，禁用模块对自己发出的每个DALI数据帧时返回DATT_SENDxxx_CFM响应，而只使用模块在所有数据帧发送完成后回复的DATT_SENDxxx_CFM，这样可以减少中间的数据返回次数，对采用RS485通信的应用，提高了效率。但主机需要设置较长的超时等待以避免再等待多条DALI总线数据发送的过程中判断为超时。等待的时长根据最长时间估计，如同时自动设置DTR1、DTR0、DeviceType和双次发送，模块实际需要向DALI总线发送5次16-bit数据帧，加上帧间等待（尚未考虑多主机应用的冲突重试），需要30ms*5 = 150ms。

TODO： 需要进一步区分DATT_SENDxxx_CFM、DATT_SENDxxx_IND和DATT_RECV_IND的区别：

*  DATT_SENDxxx_CFM应该用于确认发送是否成功，模块需要自行监测所有DA_TX发出的数据帧和DA_RX接收的数据帧是否吻合来判断发送的对错，在所有需要发送的数据帧均发送正确后返回异步的DATT_SENDxxx_CFM， 而DATT_SENDxxx_CFM主要通过Status字段来返回发送的状态。
* DATT_SENDxxx_IND则用于反馈自身发送的单次数据帧的具体情况，包括总线上检测到的实际数据，以及总线空闲时间。可用来做总线数据分析，一定程度上和DATT_RECV_IND并没有大的不同，只是DATT_RECV_IND在开启DALI通道的总线监控（MonitorEnable）后，接收的数据可能和主机的请求没有关系，PDU中的Seq字段为固定值。
* 因此还需要MonitorEnable标志来决定是否返回接收到的自身发送的总线数据帧（以DATT_RECV_IND或DATT_SENDxxx_IND？？？）， 参考DALI_SCI2。
* 也需要增加WaitReply标志来决定是否要等待DALI anwser，参考DALI_SCI2。
* 

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
* **BusDataBits**：要发送的数据位数。0：仅发送起始位；1~64：1~64位，其中8/16/24/32可以用来发送DALI标准的8/16/24/32位数据帧；65~255：无效。
* **BusData**：DALI总线上要发送的数据。高字节在前，低字节在后，根据BusDataBits从低位到高位依次发送。



##### DATT_SEND_CFM

Data定义为

| 1 byte  | 1 byte | 1 byte      | 0..N byte     |     1 byte      |     1 byte     |
| :-----: | :----: | ----------- | ------------- | :-------------: | :------------: |
| Channel | Status | BusDataBits | BusData[0..N] | BusIdleTimeHigh | BusIdleTimeLow |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道；1~4：通道编号；其他：保留未用。
* **Status**：发送状态， 1字节。含义参考[DATT Status](#DATT Status)说明。
* **BusDataBits**：已发送的数据位数。0：仅发送起始位；1~64：1~64位，其中8/16/24/32表示DALI标准的8/16/24/32位数据帧；65~255：无效。
* **BusData**：DALI总线上已发送的数据。高字节在前，低字节在后，根据BusDataBits从低位到高位依次发送。
* **BusIdleTimeHigh/BusIdleTimeLow**：  BusIdleTime的高低字节，参考[BusIdleTime](#BusIdleTime)。

###### DATT Status

| 名称                     | 数值 | 含义     |
| ------------------------ | ---- | -------- |
| DATT_Success             | 0    | 成功     |
| DATT_Failure             | 1    | 失败     |
| DATT_BufferFull          | 2    | 缓存区满 |
| DATT_TransmissionTimeout | 3    | 传输超时 |
| DATT_NoReply             | 4    | 无响应   |
| DATT_BusFailure          | 5    | 总线错误 |
| DATT_InvalidParameter    | 6    | 非法参数 |

###### BusIdleTime

BusIdleTime表示DALI总线发送或接收一个数据帧之前的空闲时间，取值范围0~0xFFFF。其中0~0xFFFE：表示时间长度为该数值 x 83.3us (即0.2Te，Te表示DALI标准中的1一个half-bit时间，约为416.7us )；0xFFFF：表示时间长度超过0xFFFE表示的值（约为5161ms，即5.16s）

##### DATT_RECV_IND

Data定义为

| 1 byte  | 1 byte | 1 byte      | 0..N byte     |     1 byte      |     1 byte     |
| :-----: | :----: | ----------- | ------------- | :-------------: | :------------: |
| Channel | Status | BusDataBits | BusData[0..N] | BusIdleTimeHigh | BusIdleTimeLow |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道；1~4：通道编号；其他：保留未用。
* **Status**：接收状态， 1字节。含义参考[DATT Status](#DATT Status)说明。
* **BusDataBits**：接收的数据位数。0：无数据接收；1~64：1~64位，其中8/16/24/32表示DALI标准的8/16/24/32位数据帧；65~255：无效。
* **BusData**：DALI总线上接收到的数据。高字节在前，低字节在后，根据BusDataBits从低位到高位依次接收。
* **BusIdleTimeHigh/BusIdleTimeLow**：  BusIdleTime的高低字节，参考[BusIdleTime](#BusIdleTime)。

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

##### DATT_SEND8_CFM

Data定义为

| 1 byte  | 1 byte | 1 byte |     1 byte      |     1 byte     |
| :-----: | ------ | :----: | :-------------: | :------------: |
| Channel | Status | Answer | BusIdleTimeHigh | BusIdleTimeLow |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道；1~4：通道编号；其他：保留未用。
* **Status**：发送状态， 1字节。含义参考[DATT Status](#DATT Status)说明。
* **Answer**：DALI 101 中定义的8bit后向帧（Backward Frame ）响应数值， 1字节。
* **BusIdleTimeHigh/BusIdleTimeLow**： BusIdleTime的高低字节，参考[BusIdleTime](#BusIdleTime)。

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

**TODO：增加StoreActualLevelInDTR0位。**

| bit 7         | bit 6                  | bit 5   | bit 4   | bit 3      | bit [2:0] |
| ------------- | ---------------------- | ------- | ------- | ---------- | --------- |
| SetDeviceType | StoreActualLevelInDTR0 | SetDTR1 | SetDTR0 | Send-Twice | Priority  |

数据位定义如下：

- bit 7：**SetDeviceType**， 设置DeviceType。0：不需要设置Device Type；1：自动发送Enable Device Type x命令，Device Type数值在数据帧的AdditionData2，通常用于Device Type相关的扩展命令。
- bit 6：**StoreActualLevelInDTR0**，自动发送DALI 102标准中的StoreActualLevelInDTR0命令，以便在正常命令发送之前将当前的亮度等级设置到DTR0中，一般用于配置Scene、PowerOnLevel和SystemFailureLevel等来减少一次主机交互过程。
- bit 5：**SetDTR1**，设置DTR1。0： 不需要额外发送DTR1；1：自动发送DTR1命令，DTR1的数值在数据帧的AdditionData1中，通常用于需要同时指定DTR0和DTR1的命令。
- bit 4：**SetDTR0**，设置DTR0。0： 不需要额外发送DTR0；1：自动发送DTR0命令，DTR0的数值在数据帧的AdditionData0中，通常用于STORE_XXX配置命令。
- bit 3：**Send-Twice**，两次发送。0： 单次发送；1：发送2次命令（通常用于DALI命令中的Send-Twice命令）
- bit [2:0]：**Priority**, 传输优先级，默认为0，仅用于支持multi-master的应用中。0： 标准；1~5： 对应DALI标准中的Priority 0~4。6~7：保留。

##### DATT_SEND16_CFM

Data定义为

| 1 byte  | 1 byte | 1 byte  | 1 byte |     1 byte      |     1 byte     |
| :-----: | ------ | :-----: | :----: | :-------------: | :------------: |
| Channel | Status | Address | Opcode | BusIdleTimeHigh | BusIdleTimeLow |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道（默认）；1~4：通道编号；其他：保留未用。
* **Status**：发送状态， 1字节。含义参考[DATT Status](#DATT Status)说明。
* **Address**：DALI 102 控制装置地址， 1字节。数值为DALI 102 16 bit命令帧中的Address字节。
* **Opcode**：DALI 102 控制装置操作码， 1字节。数值为DALI 102 16 bit命令帧中的Opcode字节。
* **BusIdleTimeHigh/BusIdleTimeLow**： BusIdleTime的高低字节，参考[BusIdleTime](#BusIdleTime)。

#### DALI 103 控制装置透传命令

##### DATT_SEND24

在指定DALI通道所在的总线上发送24bit前向帧（Forward Frame) 命令，主要用于和DALI 103控制设备（ControlDevice）通信。

Data定义为：

| 1 byte  | 1 byte  | 1 byte  | 1 byte | 1 byte        | 1 byte        | 1 byte        |
| ------- | ------- | ------- | ------ | ------------- | ------------- | ------------- |
| Channel | Control | Address | OpCode | AdditionData0 | AdditionData1 | AdditionData2 |

其中：

* **Channel**：DALI 通道，1字节。0：所有通道（默认）；1~4：通道编号；其他：保留未用。
* **Control**： 控制标志， 1字节。
* **Address**：DALI 102 控制装置地址， 1字节。
* **OpCode**：DALI 102 控制装置操作码， 1字节。
* **AdditionData0**：附加数据字节0。
* **AdditionData1**：附加数据字节1。
* **AdditionData2**：附加数据字节2。

**控制标志 Control **

| bit 7 | bit 6 | bit 5      | bit 4   | bit 3         | bit [2:0] |
| ----- | ----- | ---------- | ------- | ------------- | --------- |
| 保留  | 保留  | Send-Twice | SetDTR0 | SetDeviceType | Priority  |

数据位定义如下：

- bit 7：保留未用，默认为0。
- bit 6：**Priority**, 传输优先级，仅用于支持multi-master的应用中。0： 标准（默认）；1~5： 对应DALI标准中的Priority 0~4。
- bit 5：**Send-Twice**，两次发送。0： 单次发送；1：发送2次命令（通常用于DALI命令中的Send-Twice命令）
- bit 4：**SetDTR0**，设置DTR0。0： 不需要额外发送DTR0；1：自动发送DTR0命令，DTR0的数值在数据帧的AdditionData0中，通常用于STORE_XXX配置命令。
- bit 3：**SetDeviceType**， 设置DeviceType。0：不需要设置Device Type；1：自动发送Enable Device Type x命令，Device Type数值在数据帧的AdditionData，通常用于Device Type相关的扩展命令。
- bit [2:0]：**Priority**, 传输优先级，默认为0，仅用于支持multi-master的应用中。0： 标准；1~5： 对应DALI标准中的Priority 0~4。6~7：保留。

##### DATT_SEND24_CFM

Data定义为

| 1 byte  | 1 byte | 1 byte  |  1 byte  | 1 byte |     1 byte      |     1 byte     |
| :-----: | ------ | :-----: | :------: | :----: | :-------------: | :------------: |
| Channel | Status | Address | Instance | Opcode | BusIdleTimeHigh | BusIdleTimeLow |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道（默认）；1~4：通道编号；其他：保留未用。
* **Status**：发送状态， 1字节。含义参考[DATT Status](#DATT Status)说明。
* **Address**：DALI 103 控制设备地址， 1字节。数值为DALI 103 24 bit命令帧中的Address字节。
* **Instance**：DALI 103 控制设备实例，1字节。数值为DALI 103 24 bit命令帧中的Instance字节。
* **Opcode**：DALI 103 控制设备操作码，1字节。数值为DALI 103 24 bit命令帧中的Opcode字节。
* **BusIdleTimeHigh/BusIdleTimeLow**：  BusIdleTime的高低字节，参考[BusIdleTime](#BusIdleTime)。

#### DALI 105 固件更新透传命令

##### DATT_SEND32

在指定DALI通道所在的总线上发送32bit前向帧（Forward Frame) 命令，主要用于DALI 105固件更新功能的通信。

Data定义为：

| 1 byte  | 1 byte  | 1 byte  | 1 byte  | 1 byte  |
| ------- | ------- | ------- | ------- | ------- |
| Channel | Address | Opcode1 | Opcode2 | Opcode3 |

其中：

* **Channel**：DALI 通道，1字节。0：所有通道（默认）；1~4：通道编号；其他：保留未用。
* **Address**：DALI 105 地址字节， 1字节。
* **Opcode1/2/3**：DALI 105 操作码， 3字节。

##### DATT_SEND32_CFM

Data定义为

| 1 byte  | 1 byte | 1 byte  | 1 byte  | 1 byte  | 1 byte  |     1 byte      |     1 byte     |
| :-----: | ------ | :-----: | :-----: | ------- | :-----: | :-------------: | :------------: |
| Channel | Status | Address | Opcode1 | Opcode2 | Opcode3 | BusIdleTimeHigh | BusIdleTimeLow |

其中，

* **Channel**：DALI 通道，1字节。0：所有通道（默认）；1~4：通道编号；其他：保留未用。
* **Status**：发送状态， 1字节。含义参考[DATT Status](#DATT Status)说明。
* **Address**：DALI 105 地址字节， 1字节。
* **Opcode1/2/3**：DALI 105 操作码， 3字节。
* **BusIdleTimeHigh/BusIdleTimeLow**：  BusIdleTime的高低字节，参考[BusIdleTime](#BusIdleTime)。

#### DALI 透传命令应用示例

##### 102控制装置示例

以下实例以DALI通道1为例，所有Channel为0x01。



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

1. SyncResponse：DEFAULT_RSP（默认应答帧），表示已正确接收并开始执行命令；
2. AsyncReport：DATT_SEND16_CFM（已发送16bit数据指示帧），汇报发送状态。

3. AsyncReport：DATT_RECV_IND，指示无需DALI总线响应数据。



**主机查询地址0x01的控制装置状态 **

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

1. SyncResponse：DEFAULT_RSP，表示已正确接收并开始执行命令；
2. AsyncReport：DATT_SEND16_CFM，汇报发送状态。
3. AsyncReport：DATT_RECV_IND，指示总线上接收到的响应帧（正常情况下为8 bit 后向帧数据）或DALI总线在标准允许的时间内未接收到响应帧。



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

1. SyncResponse：DEFAULT_RSP，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_SEND16_CFM，汇报已发送“DTR0”（AdditionData0）命令。
3. AsyncReport：DATT_SEND16_CFM，汇报已发送第1次“SET POWER ON LEVEL (DTR0)”命令。
4. AsyncReport：DATT_SEND16_CFM，汇报已发送第2次“SET POWER ON LEVEL (DTR0)”命令。
5. AsyncReport：DATT_RECV_IND，指示无需总线响应。



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

1. SyncResponse：DEFAULT_RSP，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_SEND16_CFM，汇报已发送“DTR0”（AdditionData0）命令。
3. AsyncReport：DATT_SEND16_CFM，汇报已发送第1次“ENABLE WRITE MEMORY”命令。
4. AsyncReport：DATT_SEND16_CFM，汇报已发送第2次“ENABLE WRITE MEMORY”命令。
5. AsyncReport：DATT_RECV_IND，指示无需总线响应。

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

1. SyncResponse：DEFAULT_RSP，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_SEND16_CFM，汇报已发送“DTR1（data）”命令。
3. AsyncReport：DATT_RECV_IND，指示无需总线响应。

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

1. SyncResponse：DEFAULT_RSP，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_SEND16_CFM，汇报已发送“WRITE MEMORY LOCATION(DTR1,DTR0, data)”命令。
3. AsyncReport：DATT_RECV_IND，指示无需总线响应。



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

1. SyncResponse：DEFAULT_RSP，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_SEND16_CFM，汇报已发送“DTR1”（AdditionData1）命令。
3. AsyncReport：DATT_SEND16_CFM，汇报已发送“DTR0”（AdditionData0）命令。
4. AsyncReport：DATT_SEND16_CFM，汇报已发送“READ MEMORY LOCATION(DTR1,DTR0) ”命令。
5. AsyncReport：DATT_RECV_IND，指示总线上接收到的响应帧（正常情况下为8 bit 后向帧数据）或DALI总线在标准允许的时间内未接收到响应帧。



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

1. SyncResponse：DEFAULT_RSP，表示已正确接收并开始执行命令。
2. AsyncReport：DATT_SEND16_CFM，汇报已发送“DTR0”（AdditionData0）命令。
3. AsyncReport：DATT_SEND16_CFM，汇报已发送“ENABLE DEVICE TYPE X”（AdditionData2）命令。
4. AsyncReport：DATT_SEND16_CFM，汇报已发送第1次“STORE DTR AS FAST FADE TIME”命令。
5. AsyncReport：DATT_SEND16_CFM，汇报已发送第2次“STORE DTR AS FAST FADE TIME”命令。
6. AsyncReport：DATT_RECV_IND，指示无需总线响应。

##### 103控制设备示例

待添加。

### DALI 应用命令接口

CmdCat为 0x04。

DALI 应用命令接口（DALI Application Command Interface）用来对底层DALI数据传输进行封装，为上层业务提供简单的调用接口。使用该接口可以减少主机和模块的交互次数从而降低主机对DALI底层传输指令的控制需求，也可以提高通信效率改善用户体验，当然和透传接口相比，无法做到对DALI总线数据传输细节的监控。这些接口命令包括：

| Direction | FrameType    | 名称                      | CmdId | 描述                       |
| --------- | ------------ | ------------------------- | ----- | -------------------------- |
| 0         | SyncRequest  | **DAA_BPS_CTRL**          | 0x00  | DALI总线电源控制           |
| 0         | SyncRequest  | **DAA_BPS_STAUS**         | 0x01  | DALI总线电源状态获取       |
| 0         | AsyncRequest | **DAA_CG_DISC**           | 0x02  | DALI控制装置设备搜索       |
| 1         | AsyncReport  | **DAA_CG_DISC_IND**       | 0x82  | DALI控制装置设备搜索指示   |
| 0         | AsyncRequest | **DAA_CG_ADDRESSING**     | 0x03  | DALI控制装置地址分配       |
| 1         | AsyncReport  | **DAA_CG_ADDRESSING_IND** | 0x83  | DALI控制装置地址分配指示   |
| 0         | AsyncRequest | **DAA_CG_CTRL**           | 0x10  | DALI控制装置控制           |
| 0         | AsyncRequest | **DAA_CG_CFG**            | 0x11  | DALI控制装置配置           |
| 0         | AsyncRequest | **DAA_CG_QUERY**          | 0x12  | DALI控制装置查询           |
| 0         | AsyncRequest | **DAA_CG_MB_READ**        | 0x13  | DALI控制装置MemoryBank读取 |
| 0         | AsyncRequest | **DAA_CG_MB_WRITE**       | 0x14  | DALI控制装置MemoryBank写入 |
| 0         | AsyncRequest | **DAA_CG_APP_CTRL**       | 0x15  | DALI控制装置扩展控制       |
| 0         | AsyncRequest | **DAA_CG_APP_CFG**        | 0x16  | DALI控制装置扩展配置       |
| 0         | AsyncRequest | **DAA_CG_APP_QUERY**      | 0x17  | DALI控制装置扩展查询       |

#### DALI 总线电源管理应用接口

##### DAA_BPS_CTRL

##### DAA_BPS_STATUS

#### DALI 控制装置应用接口

##### DALI 控制装置搜索

主机发送DAA_CG_DISC命令请求模块开始搜索DALI总线上的102控制装置，模块执行搜索并返回DAA_CG_DISC_IND数据帧包括：

* 总线是否有设备在线；
* 总线上是否有未分配地址的设备；
* 总线上已分配地址的设备短地址（Short Address）列表。

指示搜索DALI总线过程中的结果，可在不同的阶段汇报多次结果。

###### DAA_CG_DISC

DAA_CG_DISC 命令PDU的data部分内容为：

| 1 byte  |      1 byte      |
| :-----: | :--------------: |
| Channel | DiscoveryOptions |

其中，

* Channel：DALI 通道编号。
* DiscoveryOptions：搜索选项。

**DiscoveryOptions**

|   bit 7:4   | bit 3                  | bit 2        | bit 1                    | bit 0           |
| :---------: | ---------------------- | ------------ | ------------------------ | --------------- |
| ExpiredTime | RequireQueryDeviceType | ForceRestart | NoIntermediateIndication | UnaddressedOnly |

其中，

* ExpiredTime：取值0~15，超时时间。单位：分钟。0表示没有超时限制。
* RequireQueryDeviceType：需要查询设备类型。0：不需要，1：需要，此处查询仅返回单个DeviceType类型或者指示存在多个设备类型，多个设备类型仍需要主机发起独立查询请求来获取控制装置支持的所有设备类型列表。
* ForceRestart：如果有未完成的搜索，是否强制重启搜索。0：否，等待当前搜索完成；1：是，强制结束当前未完成的搜索，重新启动搜索。
* NoIntermediateIndication：不需要搜索过程中汇报进度。0：不汇报，直到搜索完成后才汇报搜索结果；1：搜索过程中汇报搜索状态变化。
* UnaddressedOnly：只搜索是否存在未分配地址的设备。0：否，搜索所有设备，1：是，仅搜索未分配地址的设备。

###### DAA_CG_DISC_IND

DAA_CG_DISC_IND 数据帧用于汇报设备搜索结果，如果DAA_CG_DISC中指定了DiscoveryOptions的NoIntermediateIndication为1，则在搜索过程中每当发现新的设备，模块都会发送DAA_CG_DISC_IND 用于汇报状态。否则，模块只有在结束搜索后才发送该数据帧进行汇报。

如果DAA_CG_DISC中DiscoveryOptions的ExpiredTime不为0，则模块搜索过程会在超出ExpiredTime时异常结束。

如果DAA_CG_DISC中DiscoveryOptions的RequireQueryDeviceType为1，则模块搜索过程进行Query DeviceType，并将结果包含在DAA_CG_DISC_IND中。

DAA_CG_DISC_IND 数据帧的PDU data定义如下：

| 1 byte  |     1 byte      |    0.. N byte(s)     |
| :-----: | :-------------: | :------------------: |
| Channel | DiscoveryStatus | DiscoveredDeviceList |

其中，

* Channel：DALI 通道编号。
* Status：搜索状态指示。含义参考后文的DiscoveryStatus说明。
* DiscoveredDeviceList：搜索到的设备列表，具体格式参考DiscoveredDeviceList说明。

**DiscoveryStatus**

|       bit 7       |         bit 6          | bit 5:4  |    bit 3:0     |
| :---------------: | :--------------------: | -------- | :------------: |
| DeviceTypeQueried | UnaddressedDeviceFound | 保留未用 | DiscoveryState |

其中，

* DeviceTypeQueried：包含DeviceType。
* UnaddressedDeviceFound：发现未分配地址的设备。
* State：搜索状态，取值0~15。0： 正常已完成；1：进行中；2：超时结束；3：异常结束；4~15：保留 。

**DiscoveredDeviceList**

|   1 byte   |     0.. M byte(s)      |
| :--------: | :--------------------: |
| ListLength | DiscoveredDeviceRecord |

其中，

* ListLength：列表长度。
* DiscoveredDeviceRecord： 已发现设备的记录，参考DiscoveredDeviceRecord格式说明。

**DiscoveredDeviceRecord**

|    1 byte    |   1 byte   |
| :----------: | :--------: |
| ShortAddress | DeviceType |

其中，

* ShortAddress：0~63表示已分配地址的设备短地址；255（0xFF）表示存在未分配地址的设备。
* DeviceType：设备类型。0~223： 设备类型， 目前DALI标准中实际定义启用的设备类型仅为一部分；224~253：保留未用；254：没有支持任何扩展设备类型， 或者当DiscoveryStatus中的DeviceTypeQueried为0时；255：支持多个扩展设备类型。

##### DAA_CG_ADDRESSING

##### DAA_CG_CTRL

##### DAA_CG_CFG

##### DAA_CG_QUERY

##### DAA_CG_MB_READ

##### DAA_CG_MB_WRITE

##### DAA_CG_APP_CTRL

##### DAA_CG_APP_CFG

##### DAA_CG_APP_QUERY

#### DALI 控制设备应用接口

暂未定义。



## 主机侧完整应用示例

