# ToshibaLineController
ESP8266(ESPHOME) Based Line Controller for Toshiba air conditioner.
Decode Toshiba AB protocol for air conditioners with wired controllers.

Tested with remote control unit RBC-ASC11E.

- 线控协议及硬件电路参考了以下项目：
  - https://github.com/issalig/toshiba_air_cond

# 项目背景
东芝中央空调的室内机和室外机之间（室内机的U1/U2端子），可能采用了HBS（Home Bus System）协议，这里我没有集控网关，无法进行分析。看到了toshiba_air_cond(https://github.com/issalig/toshiba_air_cond) 这个项目之后，我开始参考这一项目，尝试在室内机的A/B端子，即连接到线控器的线缆上，增加一个运行ESPHOME固件的ESP8266设备，用于接入HomeAssistant，进行远程控制和状态读取。

在issalig的项目中，他描述了A/B端子的物理层协议，以及逆向分析出的控制协议等，给了我很大帮助。

但是根据issalig的描述，他的线控器A/B端子间电压高电平为15.6V，低电平为14V。和我这里测量到的电平有所不同。根据我的测试结果，随着负载电流增大，A/B端子的高电平电压会降低。因此靠稳压二极管可能没办法很好的适应不同负载的情况，因为我尝试使用LM393电压比较器，来搭建一个不受电压影响的电压比较电路。

# 物理层协议
东芝的A/B端子间，我实测开路电压为17.6V左右，连接一个线控器（工作电流30mA)后，端子电压为16.7V，打开线控器背光（工作电流50mA)后，端子电压降为16V左右。因此，我推测室内机内部应该是如下图的结构：一个18V（大概）的直流电源，具有比较大的内阻（比如100R）。当主机不发送数据时，总线电压为正常的电源电压，即18V，定义此状态为逻辑“1”。当总线发送逻辑“0”时，会将总线上加入一个负载，使得总线电流增大，由于电源有较大的内阻，所以路端电压随之下降（比如降低1V，为17V）。从机（线控器）同理。
![image](https://user-images.githubusercontent.com/11470789/123516352-3a9e6c00-d6ce-11eb-9339-bb1e90c46cd4.png)

# 串口协议
上述A/B端子总线上，数据传输使用UART协议，半双工，波特率2400，8数据位，偶校验。协议细节参考issalig的项目。
我的主机地址是01而不是和issalig一样的00。这里按照我的主机协议，重新整理协议如下：

数据包格式如下:

|Source | Dest | Opcode 1  | Data Length | Data | XOR8 |
|---|---|---|---|---|---|

数据区由以下部分组成:

| R/W mode | Opcode 2 | Payload |
|---|---|---|




Source (1 byte): 
|#|Desc|
|---|---|
|01 | 主机 (室内机) |
|40 | 线控器 |
|FE | 广播 |
|52 |未知|

Dest (1 byte):
|#|Desc|
|---|---|
|01 | 主机 (室内机) |
|40 | 线控器 |
|FE | 广播 |
|52 |未知|

Operation code 1 (1 byte) 
- From master (00)
  |Opc1|Desc|Example|
  |---|---|---|
  |10| ping|    00 FE 10 02 80 8A E6|
  |11| parameters (temp. power, mode, fan, save)| 00 52 11 04 80 86 84 01 C4| 
  |1A| sensor value| 00 40 1A 07 80 EF 80 00 2C 00 2B B5|
  |1C| status |00 FE 1C 0D 80 81 8D AC 00 00 76 00 33 33 01 00 01 B9|
  |58| extended status | |
  |18| pong, answer to remote ping |00 40 18 08 80 0C 00 03 00 00 48 00 97|
  |18| master ack after setting param  |00 40 18 02 80 A1 7B|
- From remote (40)
  |Opc1|Desc|Example|
  |---|---|---|
  |11|  set power |  40 00 11 03 08 41 03 18 |
  |11 | set mode |   40 00 11 03 08 42 01 19 (heat)|
  |11 | set temp | 40 00 11 08 08 4C 09 1D 6C 00 05 05 65|
  |11 | set save | |
  |15| Error history   | 40 00 15 07 08 0C 81 00 00 48 00 9F  |
  |17|   sensor query   | 40 00 17 08 08 80 EF 00 2C 08 00 02 1E |
  | 55        temperature|    40 00 55 05 08 81 00 69 00 F0 |
  

Data length (1 byte)
- Length of data field

Data (composed of the following parts)

R/W mode (1 byte)
|Mode|Desc|
|---|---|
|08 |for Write mode (from remote 40)|
|80 |for Read mode (from master 00)|

Opcode2

  - From master (00)
  
    | Opcode2 | Desc | Example |
    |---|---|---|
    | 81 | status | 00 FE 1C 0D 80 81 34 A8 00 00 6C 00 55 55 01 00 01 1E |
    | 81 | extended status | 00 FE 58 0F 80 81 35 AC 02 00 6E 6F E9 00 55 55 01 00 01 DB|
    | 8A | alive   |00 FE 10 02 80 8A E6|
    | A1 | ack after setting param  |00 40 18 02 80 A1 7B|
    | 86 | mode  |00 52 11 04 80 86 24 05 60| 
    | 0C | pong,answer to ping opc2 0C | 00 40 18 08 80 0c 00 03 00 00 48 00 97 | 

  - From remote (40)
  
  
    | Opcode2 | Desc | Example |
    |---|---|---|
    | 41 |power|40 00 11 03 08 41 02 19|
    | 42 |mode|40 00 11 03 08 42 02 1A|
    | 4C |temp, fan|40 00 11 08 08 4C 11 1A 6E 00 55 55 78|
    | 54 |save (opc1 11)|40 00 11 04 08 54 01 00 08 |
    | 80 |sensor query|40 00 17 08 08 80 EF 00 2C 08 00 02 1E|
    | 81 |sensor room temp |40 00 55 05 08 81 00 6A 00 F3|
    | 0C 81 | ping |40 00 15 07 08 0C 81 00 00 48 00 9F|
    | 0C 82 |timer |40 00 11 09 08 0C 82 00 00 30 05 01 01 EB|
    
    

CRC is computed as Checksum8 XOR of all the bytes (Compute it at https://www.scadacore.com/tools/programming-calculators/online-checksum-calculator/)

# Message types

There are two different status messages sent from master: normal and extended. Opcode for both is 81
Extended has two extra bytes, one could be some temperature? and other is always E9 in my experiments.

Normal status
```
00 FE 1C 0D 80 81 8D AC 00 00 76 00 33 33 01 00 01 B9
|  |     ||       |  |         |    |      |- save mode bit0
|  |     ||       |  |         |    |  |- bit2 HEAT:1 COLD:0 
|  |-Dst |        |  |         |    |- bit2 HEAT:1 COLD:0
|-Src    |        |  |         |- bit7..bit1  - 35 =Temp
         |        |  |-bit7..bit5 fan level (auto:010 med:011 high:110 low:101 )
         |        |  |-bit2 ON:1 OFF:0
         |        |-bit7.bit5 (mode cool:010 fan:011 auto 101 heat:001 dry: 100)
         |        |-bit0 ON:1 OFF:0
         |-Byte count
         
remote, last byte bit0
master, status in two bits  byte bit0  byte bit2
```
Extended status 
```
                                 -- -- extra values in extended
00 FE 58 0F 80 81 8D A8 00 00 7A 84 E9 00 33 33 01 00 01 9B
                                    |-always E9
                                 |-  1000 0100  1000010 66-35=31 (real temp??)  
                              |-temp 0111 1010 111101 61-35 = 26    
                        |- 0x2 pre-heat
 
temperature reading also confirmed  in pg14 https://www.toshibaheatpumps.com/application/files/8914/8124/4818/Owners_Manual_-_Modbus_TCB-IFMB640TLE_E88909601.pdf                                                      
``` 

ALIVE message (sent every 5 seconds from master)
```
00 fe 10 02 80 8a e6 
|  |  |  |  |  |  |- CRC
|  | OPC1|  |  |- OPC2 
|  ALL   |  |-from master, info message??
Master   |- Length

From master (00) to all (fe)

```

ACK sent after setting parameters
```
00 40 18 02 80 a1 7b 
```

PING/PONG, remote pings and master pongs
```
40 00 15 07 08 0c 81 00 00 48 00 9f 
               |- opcode ping
               
00 40 18 08 80 0c 00 03 00 00 48 00 97
               |- opcode ping
```

Temp from remote to master.
```
40 00 55 05 08 81 00 68 00 f1 
      |           |  |- 0110 1000 -> 104/2 -> 52 - 35 = 17   (temp)
      |-opc1         |-bit0 ON:1 OFF:0                 
      
```

From master mode status Opcode 2 0x86
```
00 52 11 04 80 86 24 00 65  heat
      |-opc1      |  |- mode  bit7-bit5, power bit0, bit2 ???
                  |- 0010 0100 -> mode bit7-bit5  bit4-bit0 ???
                     ---                  
```

Setting commands

Power
```
40 00 11 03 08 41 03 18    ON
40 00 11 03 08 41 02 19   OFF
                  |- power >> 1 | 0b1
```                  
Set mode
```
40 00 11 03 08 42 01 19 //heat
                  |-(Mode is bit3-bit0 from last data 6th byte) cool:010 fan:011 auto 101 heat:001 dry: 100            
```

Set temperature
```
00 01 02 03 04 05 06 07 08 09 10 11 12  byte nr
40 00 11 08 08 4C 0C 1D 7A 00 33 33 76
                              |  |-0x33 for cold 0x55 for heat
                              |-0x33 for cold 0x55 for heat
                  |  |  |- ((target_temp) + 35) << 1
                  |  |- fan | 0b11000 
                  |- mode | 0b1000  cool:010 fan:011 auto 101 heat:001 dry: 100
```

Set fan (requires info about target temp, fan and heat/cold mode)

```
00 01 02 03 04 05 06 07 08 09 10 11 12 byte nr
40 00 11 08 08 4C 13 1D 7A 00 33 33 6E //LOW, FAN
                  |  |  |     |  |-0x33 for cold 0x55 for heat
                  |  |  |     |- 0x33 for cold 0x55 for heat
                  |  |  |- (target_temp) + 35) << 1
                  |  |- fan bit4=1 bit3-bit1  auto:0x010 med:011 high:110 low:101
                  |-  0x10 + mode  cool:010 fan:011 auto 101 heat:001 dry: 100
```

Set save
```
40 00 11 04 08 54 01 01 09  Save ON
40 00 11 04 08 54 01 00 08  Save OFF
                     |- bit 0 7th byte
```
TEST+SET for Error history
```
40 00 15 03 08 27 01 78
                  |-Error 1
00 40 18 05 80 27 08 00 48 ba
                        |-Type 0x4 Num error 0x8   E-08

```

TEST + CL for sensor query
```
40 00 17 08 08 80 EF 00 2C 08 00 F3 EF   
                                 |- Filter sign time
00 40 1A 07 80 EF 80 00 2C 03 1E 83  
                           |  |
                           |--|------0x03 0x1E->  798  2bytes

00 40 1A 05 80 EF 80 00 A2 12  answer for unknown sensor

```

Timer is decoded but remote takes care of it internally and ignores it if sent from external sources (our circuit)

```
40 00 11 09 08 0c 82 00 00 30 07 02 02 e9    1h for poweron
                                    |----- number of 30 minutes periods,  2 -> 1h
                                 |----- number of 30 minutes periods,  2 -> 1h
                              |------ 07 poweron   06 poweroff repeat 05 poweroff  00 cancel
                              

Sequence observed for 1h poweron

40 00 11 03 08 41 03 18    powers on
00 40 18 02 80 a1 7b       ack
40 00 11 09 08 0c 82 00 00 30 00 04 01 eb  cancels timer
00 40 18 02 80 a1 7b       ack

```
# Sensor addresses

| No.  | Desc  | Example value  |
|---|---|---|
| 00 | Room Temp (Control Temp) (°C) | Obtained from master status frames 00 FE 1C ...|
| 01 | Room temperature (remote controller) | Obtained from controller messages 40 00 55 ... |
| 02 | Indoor unit intake air temperature (TA) | 23 |
| 03 | Indoor unit heat exchanger (coil) temperature (TCJ) Liquid | 19 |
| 04 | Indoor unit heat exchanger (coil) temperature (TC) Vapor | 19 |
| 07 | Indoor Fan Speed|  0 |
| 60 | Outdoor unit heat exchanger (coil) temperature (TE) | 18 |
| 61 | Outside air temperature (TO)| 19 |
| 62 | Compressor discharge temperature (TD) | 33 |
| 63 | Compressor suction temperature (TS) | 26 |
| 65 | Heatsink temperature (THS) | 55 |
| 6a | Operating current (x1/10) | 0 |
| 6d | TL Liquid Temp (°C) | 22 |
| 70 | Compressor Frequency (rps)| 0 |
| 72 | Fan Speed (Lower) (rpm) | 0 |
| 73 | Fan Speed (Upper) (rpm) | defined in manual, not working  |
| 74 | ? | 43, is this fan speed upper? |
| 75 | ? | 0 |
| 76 | ? | 0 |
| 77 | ? | 0 |
| 78 | ? | 0 |
| 79 | ? | 0 |
| f0 | ? | 204 |
| f1 | Compressor cumulative operating hours (x100 h) | 7 |
| f2 | Fan Run Time (x 100h) | 8 |
| f3 | Filter sign time x 1h | 37 |


# Notes from logs (used for the above info)

```
Op code from remote
4C 0C 1D  set temp     40 00 11 08 08 4C 0C 1D 78 00 33 33 74
0C 81     status      
41        power        40 00 11 03 08 41 03 18
42        mode         40 00 11 03 08 42 02 1A //cool  02 -> 0000 0010
4C 14     fan          40 00 11 08 08 4C 14 1D 7A 00 33 33 6E  //low
54        save         40 00 11 04 08 54 01 00 08
0C 82     timer
40 00 15 07 08 0c 81 00 00 48 00 9f 
00 40 18 08 80 0c 00 03 00 00 48 00 97 

Op code from master
81 status              00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5
8A ack (dest FE)       00 FE 10 02 80 8A E6 # every 5s
A1 ack (dest 40)       00 40 18 02 80 A1 7B
86 ?? (dest 52)        00 52 11 04 80 86 84 05 C0
                       00 52 11 04 80 86 84 01 C4   DRY,LOW
                       00 52 11 04 80 86 64 01 24   FAN,LOW
                       00 52 11 04 80 86 44 01 04   COOL,LOW
                       00 52 11 04 80 86 44 05 00   COOL,MED
                       
0C (ASNWER TO 0c)      00 40 18 08 80 0C 00 03 00 00 48 00 97 

UNK is
10 ack / ping??           00 FE 10 02 80 8A E6
1C status after request   00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5
58 ?? status when idle    00 FE 58 0F 80 81 8D AC 00 00 7A 84 E9 00 33 33 01 00 01 9E
18 ack (dest remote)      00 40 18 02 80 A1 7B 
11 ??                     40 00 11 03 08 41 03 18
                          00 52 11 04 80 86 84 05 C0
15 (from remote)          40 00 15 07 08 0C 81 00 00 48 00 9F
55 (from remote)          40 00 55 05 08 81 00 7E 00 E7                
                          40 00 55 05 08 81 00 7C 00 E5 (heat mode 24)

first part is 1,5
second part is 0,1,8,C


00 FE 1C 0D 80 81 CD 8C 00 00 76 00 33 33 01 00 01 D9    // CD 8C 00 -> 1100 1101 1000 1100
00 FE 10 02 80 8A E6                                                    ---         -

Status
00 FE 1C 0D 80 81 8D AC 00 00 76 00 33 33 01 00 01 B9
|  |     ||       |  |         |    |      |- save mode bit0
|  |     ||       |  |         |    |  |- bit2 HEAT:1 COLD:0 
|  |-Dst |        |  |         |    |- bit2 HEAT:1 COLD:0
|-Src    |        |  |         |- bit7..bit1  - 35 =Temp
         |        |  |-bit7..bit5 fan level (auto:010 med:011 high:110 low:101 )
         |        |  |-bit2 ON:1 OFF:0
         |        |-bit7.bit5 (mode cool:010 fan:011 auto 101 heat:001 dry: 100)
         |        |-bit0 ON:1 OFF:0
         |-Byte count
         
remote, last byte bit0
master, status in two bits  byte bit0  byte bit2

         
00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5         -> 8D AC  1000 1101 1010 1100
                   -  -                                                         -       -         
cool
00 FE 1C 0D 80 81 4D AC 00 00 76 00 33 33 01 00 01 79    // 4D AC 00 -> 0100 1101 1010 1100
                                                                        ---
00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5   -> A 1010
                     -                                       ---

Extended status
00 FE 58 0F 80 81 8D AC 00 00 7A 7A E9 00 33 33 01 00 01 60 
00 FE 58 0F 80 81 8D A8 00 00 7A 84 E9 00 33 33 01 00 01 9B
                                    |-always E9
                                 |-  1000 0100  1000010 66-35=31 (real temp??)  
                              |-temp 0111 1010 111101 61-35 = 26
                              
                              
Current temp from remote (off 27C and blow on thermistor until 30C and cold again)
3rd byte is 55
bit7..bit0 /2 - 35 =Temp
40 00 55 05 08 81 00 7C 00 E5  //7C  0111 1100   124    124/2-35 = 27
40 00 55 05 08 81 00 83 00 1A  //83  1000 0011   131    131/2-35= 30.5
40 00 55 05 08 81 00 7E 00 E7  //7E  0111 1110   126    126/2-35= 28
                     --                                       
```
```
40 00 11 08 08 4c 11 1c 6c 00 55 55 7c  //Set Fan Medium
00 40 18 02 80 a1 7b  //kind of ack
```

Communication while POWERED OFF
```
00 FE 10 02 80 8A E6 # 
00 FE 10 02 80 8A E6 # every 5s
40 00 15 07 08 0C 81 00 00 48 00 9F
00 40 18 08 80 0C 00 03 00 00 48 00 97 #inmediately answer
40 00 55 05 08 81 00 7E 00 E7 
00 FE 58 0F 80 81 8D A8 00 00 7A 84 E9 00 33 33 01 00 01 9B
00 FE 10 02 80 8A E6
00 FE 10 02 80 8A E6
```

When POWERED ON 
```
00 FE 10 02 80 8A E6 # typical answer, maybe confirmation
40 00 15 07 08 0C 81 00 00 48 00 9F
00 40 18 08 80 0C 00 03 00 00 48 00 97 #inmediate answer 
40 00 55 05 08 81 00 7E 00 E7 
00 FE 58 0F 80 81 8D AC 00 00 7A 84 E9 00 33 33 01 00 01 9E
00 FE 10 02 80 8A E6 # typical answer, maybe confirmation
00 52 11 04 80 86 84 01 C4
00 FE 10 02 80 8A E6 # typical answer, maybe confirmation
00 FE 10 02 80 8A E6 # typical answer, maybe confirmation (every 5s we see  the mesg)
00 FE 10 02 80 8A E6 # typical answer, maybe confirmation (every 5s we see  the mesg)
40 00 15 07 08 0C 81 00 00 48 00 9F
00 40 18 08 80 0C 00 03 00 00 48 00 97 #inmediate answer
40 00 55 05 08 81 00 7E 00 E7
00 FE 58 0F 80 81 8D AC 00 00 7A 84 E9 00 33 33 01 00 01 9E  #inmediate answer
```

Temp down
```
40 00 11 08 08 4C 0C 1D 78 00 33 33 74 # press temp down (current 26, 25 after pressing)
00 40 18 02 80 A1 7B # typical answer, maybe confirmation
00 FE 1C 0D 80 81 8D AC 00 00 78 00 33 33 01 00 01 B7 # looks like current status
00 52 11 04 80 86 84 01 C4
00 FE 10 02 80 8A E6 # typical answer, maybe confirmation
```

```
only messages from remote
method 1
40 00 11 08 08 4C 0C 1D 7C 00 33 33 70 //27     7C -> 0111 1100  111110  14     30    62
40 00 11 08 08 4C 0C 1D 7E 00 33 33 72 //28     7E -> 0111 1110  011111  15     31    63
40 00 11 08 08 4C 0C 1D 80 00 33 33 8C //29     80 -> 1000 0000  100000   0 ???       64
                                                      ---- ---
40 00 11 08 08 4C 0C 1D 7C 00 33 33 70 //27     7C -> 0111 1100  111110  14           62
40 00 11 08 08 4C 0C 1D 7A 00 33 33 76 //26     7A -> 0111 1010  111101  13           61
40 00 11 08 08 4C 0C 1D 6E 00 33 33 62 //20     6E -> 0110 1110  110111               55
40 00 11 08 08 4C 0C 1D 6A 00 33 33 66 //18     6A -> 0110 1010  110101               53  - 35 = 18

Full log
40 00 11 08 08 4C 0C 1D 78 00 33 33 74 //25     78 -> 0111 1000  111100               60
00 40 18 02 80 A1 7B # typical answer, maybe ack
00 FE 1C 0D 80 81 8D AC 00 00 78 00 33 33 01 00 01 B7  //78

00 52 11 04 80 86 84 05 C0 #this seq was in temp up ???

00 FE 1C 0D 80 81 8D AC 00 00 76 00 33 33 01 00 01 B9


```

Power
```
remote, last byte bit0
master, status in two bits  byte bit0  byte bit2

ON
40 00 11 03 08 41 03 18    ->03 0000 0011
                   -                    -
00 40 18 02 80 A1 7B # typical answer, maybe confirmation
00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5         -> 8D AC  1000 1101 1010 1100
                   -  -                                                         -       -
00 FE 10 02 80 8A E6 
00 52 11 04 80 86 84 05 C0   -> 5 0101
                      -            - -

OFF
40 00 11 03 08 41 02 19   ->02 0000 0010
                   -                   - 
00 40 18 02 80 A1 7B 
00 FE 1C 0D 80 81 8C A8 00 00 7A 00 33 33 01 00 01 B0         -> 8C A8  1000 1100 1010 1000
00 FE 10 02 80 8A E6                                                            -       -
00 52 11 04 80 86 84 00 C5   -> 0 0000
                      -            - -
```

Modes
```
From remote (Mode is bit3-bit0 from last data byte) cool:010 fan:011 auto 101 heat:001 dry: 100
40 00 11 03 08 42 02 1A //cool  02 -> 0000 0010
40 00 11 03 08 42 03 1B //fan   03 -> 0000 0011
40 00 11 03 08 42 05 1D //auto  05 -> 0000 0101
40 00 15 02 08 42 1D    //???
40 00 11 03 08 42 01 19 //heat  01 -> 0000 0001
40 00 11 03 08 42 04 1C //dry   04 -> 0000 0100


Full log for mode changing (Mode in master is bit3-bit1 from byte 6 (starting from 0)
cool
40 00 11 03 08 42 02 1A                                  //cool   02 -> 0000 0010
00 40 18 02 80 A1 7B                                                          ---
00 FE 1C 0D 80 81 4D AC 00 00 76 00 33 33 01 00 01 79    // 4D AC 00 -> 0100 1101 1010 1100
                                                                        ---
fan
40 00 11 03 08 42 03 1B                                  //fan    03 -> 0000 0011
00 40 18 02 80 A1 7B                                                          ---
00 52 11 04 80 86 64 01 24 
00 FE 1C 0D 80 81 6D AC 00 00 76 00 33 33 01 00 01 59    // 6D AC 00 -> 0110 1101 1010 1100
00 52 11 04 80 86 64 01 24                                              --- 
00 52 11 04 80 86 64 01 24                               // 64 -> 0110 0100       new value here is 3 
                                                                  ---
auto
40 00 11 03 08 42 05 1D                                  //auto  05 -> 0000 0101
00 40 18 02 80 A1 7B                                                         ---
40 00 15 02 08 42 1D                                     
00 40 18 04 80 42 05 06 9D                               //requested 05, assigned 06??
00 FE 1C 0D 80 81 CD 8C 00 00 76 00 33 33 01 00 01 D9    // CD 8C 00 -> 1100 1101 1000 1100
00 FE 10 02 80 8A E6                                                    ---         -
                                                                        mode      fan level bit3-bit1 -> 100 ???
00 52 11 04 80 86 C4 01 84                               // C4 -> 1100 0100       
                                                                  ---                                                                                                                       
heat
40 00 11 03 08 42 01 19                                  //heat  01 -> 0000 0001
00 40 18 02 80 A1 7B                                                         ---
00 FE 1C 0D 80 81 35 AC 02 00 76 00 55 55 01 00 01 03    // 35 AC 02 -> 0011 0101 1010 1100 0000 0010   // 55 55 vs 33 33 in other modes
                                                                        ---                        -       0101 0101    0011 0011
00 52 11 04 80 86 24 01 64                               //24 -> 0010 0100
                                                                 ---
dry
40 00 11 03 08 42 04 1C                                  //dry   04 -> 0000 0100
00 40 18 02 80 A1 7B                                                         ---
00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5    // 8D AC 00 -> 1000 1101 1010 1100
                                                                        ---
00 52 11 04 80 86 84 01 C4                               // 84 -> 1000 0100
00 FE 10 02 80 8A E6                                              ---


```

Save mode

```
bit0 from 14th in master message
bit0 from 7th in remote message

40 00 11 04 08 54 01 00 08 
                      -
00 40 18 02 80 A1 7B 
00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 00 00 01 B4 
                                           -
00 52 11 04 80 86 84 01 C4 

40 00 11 04 08 54 01 01 09 
00 40 18 02 80 A1 7B 
00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5 
                                           -
00 52 11 04 80 86 84 01 C4 
```

Fan level
```
From remote request 7th byte bit3-bit1  auto:0x010 med:011 high:110 low:101
From master status  7th byte bit7-bit5

                      *
 0  1  2  3  4  5  6  7  8  9 10 11 12
40 00 11 08 08 4C 14 1A 7A 00 33 33 69                 -> A 1010  auto
                      -                                      ---
00 FE 1C 0D 80 81 8D 4C 00 00 7A 00 33 33 01 00 01 55  -> 4 0100
                     -                                      ---
...
40 00 11 08 08 4C 14 1B 7A 00 33 33 68                 -> B 1011 medium
                      -                                      ---
00 FE 1C 0D 80 81 8D 6C 00 00 7A 00 33 33 01 00 01 75  -> 6 0110
                     -                                      ---
...
40 00 11 08 08 4C 14 1C 7A 00 33 33 6F                  -> A 1100 high
                      -                                       ---
00 FE 1C 0D 80 81 8D 8C 00 00 7A 00 33 33 01 00 01 95   -> 8 1000
                     -                                       ---
...

40 00 11 08 08 4C 14 1D 7A 00 33 33 6E                  -> D 1101   low
                      -                                       ---
00 FE 1C 0D 80 81 8D AC 00 00 7A 00 33 33 01 00 01 B5   -> A 1010
                     -                                       ---
...
??
40 00 15 07 08 0C 81 00 00 48 00 9F                     -> 
               
00 40 18 08 80 0C 00 03 00 00 48 00 97 

40 00 55 05 08 81 00 7C 00 E5 
00 FE 58 0F 80 81 8D AC 00 00 7A 7D E9 00 33 33 01 00 01 67

```


TEST, ON, HEAT, COOL, OFF, TEST
```
40 00 15 07 08 0C 81 00 00 48 00 9F
40 00 55 05 08 81 00 7C 00 E5
40 00 11 03 08 41 C0 DB  test on?    1100
40 00 11 03 08 41 03 18  power on
40 00 15 07 08 0C 81 00 00 48 00 9F
40 00 11 03 08 42 02 1A
40 00 55 05 08 81 00 7C 00 E5
40 00 11 03 08 41 02 19  power off
40 00 11 03 08 41 80 9B  test off?   1000
```

TEST + CL sensor inquiry
```
40 00 17 08 08 80 EF 00 2C 08 00 02 1E
                                 |----- sensor 2
00 40 1A 07 80 EF 80 00 2C 00 15 8B  
                              |--------- 0x15 -> 21

00 40 1A 05 80 EF 80 00 A2 12  answer for unknown sensor

40 00 17 08 08 80 EF 00 2C 08 00 F3 EF   F3 Filter sign time
00 40 1A 07 80 EF 80 00 2C 03 1E 83     0x03 0x1E->  798  2bytes
```

Timer
```
40 00 11 09 08 0c 82 00 00 30 05 01 01 eb   30m poweroff
40 00 11 09 08 0c 82 00 00 30 05 02 02 eb    2h poweroff
40 00 11 09 08 0c 82 00 00 30 00 04 01 eb   cancel timer                                  
40 00 11 09 08 0c 82 00 00 30 06 03 03 e8   1.5h poweroff repeat
40 00 11 09 08 0c 82 00 00 30 06 01 01 e8   30m  poweroff repeat
40 00 11 09 08 0c 82 00 00 30 07 30 30 e9   24h poweron
40 00 11 09 08 0c 82 00 00 30 07 04 04 e9    2h poweron

40 00 11 09 08 0c 82 00 00 30 07 02 02 e9    1h for poweron
                                    |----- number of 30 minutes
                                 |----- repeated
                              |------ 07 poweron   06 poweroff repeat 05 poweroff  00 cancel

```

Power on

```
40 00 11 03 08 41 03 18                                      Power on 
00 fe 1c 0d 80 81 35 ac 00 00 6c 00 55 55 01 00 01 1b        Normal status
00 52 11 04 80 86 24 01 64                                   Mode
00 fe 10 02 80 8a e6                                         Periodic ping
40 00 15 07 08 0c 81 00 00 48 00 9f                          ??
00 40 18 08 80 0c 00 03 00 00 48 00 97                       Answer to ??
40 00 55 05 08 81 00 65 00 fc                                Sensor temp
00 fe 58 0f 80 81 35 ac 00 00 6c 6f e9 00 55 55 01 00 01 db  Extended status
00 52 11 04 80 86 24 01 64

```
Power off

```
40 00 11 03 08 41 02 19                                      Power off
00 40 18 02 80 a1 7b                                         ACK after a command
00 fe 1c 0d 80 81 34 a8 00 00 6c 00 55 55 01 00 01 1e        Normal status
00 52 11 04 80 86 24 00 65                                   Mode
00 fe 10 02 80 8a e6                                         Periodic ping
40 00 15 07 08 0c 81 00 00 48 00 9f                          ??
00 40 18 08 80 0c 00 03 00 00 48 00 97                       Answer to ??
40 00 55 05 08 81 00 6a 00 f3                                Sensor temp
00 fe 58 0f 80 81 34 a8 00 00 6c 6c e9 00 55 55 01 00 01 dd  Extended status

```
TEST+SET for Error history
```
40 00 15 03 08 27 01 78           Error 1
00 40 18 05 80 27 08 00 48 ba     Type 0x4 Num error 0x8   E-08

40 00 15 03 08 27 02 7b           Error 2
00 40 18 05 80 27 08 00 43 b1     Type 0x4 Num error 0x3   E-03

40 00 15 03 08 27 03 7a           Error 3
00 40 18 05 80 27 00 00 00 fa     0x0 0x0   No error
```

```
00 40 18 02 80 A1 7B
00 40 18 08 80 0C 00 03 00 00 48 00 97
00 40 1A 07 80 EF 80 00 2C 00 00 9E
00 52 11 04 80 86 24 00 65
00 55 55 01 00 01
00 FE 10 02 80 8A E6
00 FE 1C 0D 80 81 34 A8 00 00 6C 00 55 55 01 00 01 1E
00 FE 58 0F 80 81 34 A8 00 00 6C 6D E9 00 55 55 01 00 01 DC
40 00 11 03 08 41 02 19
40 00 11 03 08 41 03 18
40 00 11 08 08 4C 09 1D 6C 00 05 05 65
40 00 11 09 08 0C 82 00 00 30 05 01 01 EB
40 00 15 07 08 0C 81 00 00 48 00 9F
40 00 17 08 08 80 EF 00 2C 08 00 02 1E
40 00 55 05 08 81 00 66 00 FF
```

# 线控器数据接收电路（已验证）
原版线控器上，有一个LM393电压比较器，受此启发，我测试了以下接收电路，可以正常接收总线数据，同时不受总线负载情况影响。
电路原理如下：A/B端子的18V电压，由两路50K/10K电阻分压，以将电平降低到3.3V以内。其中一路直接接到比较器的正输入端，另一路通过肖特基二极管连接到比较器的负输入端。同时在二极管负极加入一个对地47uF电容，这样，当总线为高电平时，电容充电，接近分压后的高电平减去二极管压降（如2.61V-0.14V=2.47V)。总线低电平(分压后电压2.45)后，由于二极管的作用，电容电压不降低。这样，这一路电压就可以用作参考电压，当总线为逻辑1时，分压后电压为2.61V > 2.47V，比较器输出高电平；总线为逻辑0时，分压后电压为2.45V < 2.47V，比较器输出低电平。从而完成了由18V/17V到3.3V/0V的电平转换。由于二极管压降可能与比较器输入端高低电平的电压差接近，造成不正确的结果，可以调整R7/R8/R10/R11电阻的比例关系，将参考电压调整到比较器输入端压差一半的位置（(2.61V + 2.45V)/2 = 2.53V)，实现正常输出。

![image](https://user-images.githubusercontent.com/11470789/123516669-dc728880-d6cf-11eb-9e5f-ecef62c7ea25.png)

# 线控器数据发送电路（待验证）
根据A/B总线的原理，线控器需要发送逻辑0时，只要在A/B端子上增加一个负载，增大工作电流，使总线电压降低即可，这里我直接使用三极管来切换一个330Ω/1W的金属膜电阻。如以下电路：

![image](https://user-images.githubusercontent.com/11470789/123519601-ef408980-d6de-11eb-89bd-530977e79f6f.png)

# 总线供电方案（待验证）
本来考虑使用DC-DC器件实现18V到3.3V的降压转换，但使用LM2596实测发现，DC-DC器件开关过程会造成负载频繁变化，在总线上产生电压变化，干扰正常通信，使总线失效。所以我们需要使负载尽量恒定。

参考RBC-ASC11E电路板，此电路板使用了一个BA05三端稳压器，将18V左右的总线电压降低到5V。这里我们改用BA033，提供3.3V电压。


# 硬件设计
待上传


# 软件设计
待上传




