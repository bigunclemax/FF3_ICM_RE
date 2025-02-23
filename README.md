# Ford Focus 3 ICM module

There are my notes that were obtained during the reverse engineering process 
of Ford Focus 3 ICM module.  

The cluster with three gauges:  
 - Oil temperature (50-150 F)  
 - Turbo pressure (0-1 PSI)  
 - Oil pressure (0-5 PSI)  

## Connection pinout

```  
___-___
|1|2|3|
-------
|4|5|6|
-------
```
1. pin - VBAT 12V (to 16 pin of C8 BCM).
2. pin - ILL (to 14 pin of C3 BCM).
3. pin - GND
4. pin - HS-CAN-HIGH
5. pin - HS-CAN-LOW
6. pin - Not used

## Hardware
### CPU
`NXP SPC5602S (SPC5602SVLQ6)`

### I2C EEPROM
`24C16K (P51C0J)`. 2KB (16Kbit = 8 blocks x 256 bit).   
I2C addresses: `50-57`.


## Firmware
### ROM DATA

`F1ET-14G056-AA.vbf` - main program which is flashed on MCU ROM   
Offset: `0x0000c000`  
Length: `0x00034000`  

VBF header:
```C
header {
	//**********************************************************
	//*
	//*   Vector Informatik GmbH
	//*
	//*   This file was created by Hexview V1.3.0
	//*
	//**********************************************************

	//Description
	description = {"ROM DATA"
	              };

	//Software part number
	sw_part_number = "F1ET-14G056-AA";

	//Software part type
	sw_part_type = EXE;

	//Network type or list
	network = CAN_HS;

	//ecu_address or list
	ecu_address = 0x784;

	//Format frame
	frame_format = CAN_STANDARD;

	//erase block
	erase = { { 0x0000c000, 0x00034000}
	        };

	//file checksum
	file_checksum = 0xdbae8c13;

}
```

## SERVICE NVM DATA
`F1ET-14G057-AA.vbf` - Calibration data stored on EEPROM  
Offset: `0x200000d4`  
Lenght: `0x00000680`  

VBF header:
```C
header {
	//**********************************************************
	//*
	//*   Vector Informatik GmbH
	//*
	//*   This file was created by Hexview V1.3.0
	//*
	//**********************************************************

	//Description
	description = {"SERVICE NVM DATA"
	              };

	//Software part number
	sw_part_number = "F1ET-14G057-AA";

	//Software part type
	sw_part_type = EXE;

	//Network type or list
	network = CAN_HS;

	//ecu_address or list
	ecu_address = 0x784;

	//Format frame
	frame_format = CAN_STANDARD;

	//erase block
	erase = { { 0x200000d4, 0x00000680}
	        };

	//file checksum
	file_checksum = 0xae9292f4;

}
```

## CAN frames

Accepted CAN IDs:
- `7DF` - ???
- `784` - UDS protocol
- `581` - ???
- `360` - ???
- `2F0` - ???
- `130` - Mandatory for oil pressure
- `0F8` - Oil temperature, boost pressure
- `0C8` - Wakeup signal?
- `090` - RPM



### CAN ID 090
```shell
00         01         02         03         04         05         06         07
0000 0000  0000 0000  0X00 0000  0000 0000  0XXX XXXX  XXXX XXXX  0000 0000  0000 0000
                       |                     ||| ||||  |||| ||||
                       |                     ||\-----------------> Bit 36-32 | Bit 40-47 - RPM
                       |                     ||        
                       \--> Bit 22 - ???     \---> Bit 38-37 - ???
```

### CAN ID 0C8
```shell
00         01         02         03         04         05         06         07
0000 0000  0000 0XXX  XXXX X000  0000 0000  0000 0000  0000 0000  0000 0000  0000 0000
                 |||  |||| |
                 |||  |||| |
                 |||  |||| |
                 |||  \---> Bit 23-19 - ???
                 |||
                 ||\---> Bit 8 - ???
                 ||
                 \---> Bit 10-9 - ???
```

### CAN ID 0F8
```shell
00         01         02         03         04         05         06         07
0000 0000  0000 0000  0000 0000  0000 0000  XXXX XXXX  XXXX XXXX  0000 0000  XXXX XXXX
                                            |||| ||||  |||| ||||             |||| ||||
                                            |||| ||||  |||| ||||             \---------> Bit 56-63 Oil Temperature (fe-ff - move gauge to 0)
                                            |||| ||||  |||| ||||
                                            |||| ||||  |||| ||||
                                            \--------------------> Bit 47-32 Boost pressure (BE val)

```

### CAN ID 130
```shell
00         01         02         03         04         05         06         07
0000 0000  0000 0000  00XX 0000  0000 0000  0000 0000  0000 0000  0000 0000  0000 0000
                        ||
                        |\---------> Bit 20* - ???
                        |
                        \---------> Bit 21* - ???

Setting both bits sets gauge lower. Without 0x130 packet on CAN it's also low
```

### CAN ID 2F0
```shell
00         01         02         03         04         05         06         07
000X 00XX  0000 0000  0000 0000  0000 0000  0000 0000  0000 0000  0000 0000  0000 0000
   |   ||
   |   \---------> Bit 1-0 - ???
   |
   \---------> Bit 5 - ???
```

### CAN ID 360
**TODO:**

### CAN ID 581
**TODO:**

### CAN ID 784
[List of supported UDS services here](#unified-diagnostic-services-messages)

### CAN ID 7DF
**TODO:**


## Unified Diagnostic Services messages

UDS client(tester) ID - `0x784`  
UDS server(ICM ECU) ID - `0x78C`

Supported services:
- `0x10` DIAGNOSTIC_SESSION_CONTROL
- `0x11` ECU_RESET
- `0x14` CLEAR_DIAGNOSTIC_INFORMATION
- `0x19` READ_DTC_INFORMATION
- `0x00` Unknown service
- `0x22` READ_DATA_BY_IDENTIFIER
- `0x27` SECURITY_ACCESS
- `0x2e` WRITE_DATA_BY_IDENTIFIER
- `0x2f` INPUT_OUTPUT_CONTROL_BY_IDENTIFIER
- `0x31` ROUTINE_CONTROL
- `0x34` REQUEST_DOWNLOAD
- `0x36` TRANSFER_DATA
- `0x37` REQUEST_TRANSFER_EXIT
- `0x3e` TESTER_PRESENT
- `0x85` CONTROL_DTC_SETTING

### Diagnostic session control (0x10)
- `01` - Default Session
- `02` - Programming Session 
- `03` - Extended Diagnostic Session 

### ECU Reset (0x11)
- `01` - Hard Reset

### Clear Diagnostic Information (0x14)
- `FF FF FF` - clears all DTC's

### Read DTC Information (0x19)
- `01` - Report Number Of DTC By Status Mask
- `02` - Report DTC By Status Mask	
- `06` - Report DTC External Data Record By DTC Number
- `0A` - Report All Supported DTC	

### Read Data by Identifier (0x22)
- `XX XX` - DID

### Security access (0x27)
- `01` - Request Seed (Level 0)
- `02` - Send Key (Level 0)
- `21` - Request Seed (Level 2)
- `22` - Send Key (Level 2)

### Write Data By Identifier (0x2e)  
Allowed DIDs:  
- `da70`
- `da71`
- `da80`
- `da81`
- `da93`
- `da94`
- `da95`
- `dad1`
- `dad2`
- `dad3`
- `fd29`
- `fd30`
- `fd31`

See [DIDs detailed description here](#uds-dids)

### Input Output Control By Identifier (0x2F)
- `0x60 0x03  0x00` - `DID 6003` returnControlToEcu
- `0x60 0x03  0x03` - `DID 6003` shortTermAdjustemt

- `0x60 0x0d  0x00`
- `0x60 0x0d  0x03`

- `0x60 0x17  0x00`
- `0x60 0x17  0x03`

- `0x61 0xbd  0x00`
- `0x61 0xbd  0x03`

### Routine Control (0x31)
Allowed routineIdentifier:
- `0202` (0)
- `0301` (1)
- `0304` (2)
- `FF00` (3) - eraseMemory
- `FF01` (4) - checkProgrammingDependencies

Allowed sub-function:
- `01` - startRoutine (STR)
- `02` - stopRoutine (STPR)
- `03` - requestRoutineResults (RRR)

Allowed routineId&sub-func matrix:
```
			STR(0)		STPR(1) 	RRR(2)
0202 (0)		0			1			2
0301 (1)		3			7			7
0304 (2)		4			7			7
FF00 (3)		5			7			7
FF01 (4)		6			7			7

7 means N\A
```

### Request Download (0x34)

### Transfer Data (0x36)

### Request Transfer Exit (0x37)

### Tester present (0x3e)
- `00` - No sub-function value beside SuppressPosResMsgIndication (SPRMI) not supported

### Control DTC Settings (0x85)
- `01` - Turn ON
- `02` - Turn OFF

## UDS DIDs

```shell
Identified DIDs:
DID    Value (hex)
0x0202 00
0x6003 00               # Oil_PSI (Min: 0, Max: 100 - Gauge position in percents!)
0x600d 0000             # Turbo_PSI (Min: 0, Max: 0xCE)
0x6017 12c0             # Oil_temp
0x61bd 00               # Backlight_level (Min: 0, MAX: 0xFF)
0xd100 01
0xda70 001e14
0xda71 001e1d           # backlight CH 18
0xda80 001a
0xda81 001a
0xda93 ffffcc           # backlight
0xda94 1054a9f40a48a0ff # backlight
0xda95 1054a9f40a48a0ff # backlight

# Turbo_PSI table (len 36)
0xdad1 00001999000000001999fffaffffffffffffffffffffffffffffffffffffffffffffffff
# Oil_PSI table (len 450)
0xdad2 0000006400b601a60207029702f703e803e803e803e803e803e803e803e80000038703e8048a04e0056405dc06fe07940834088408980898089808980000087a099a09e70a280a810b370cd40d770df40e410e7a0ea10ea10ec700000d810f6b102b10d7118e1261133e139e13d8144114b414ee14f8151e0000100e113011c6125c12f2138814f815c116041648167616a816a816a80000106e10bb1130121412cb1388150115cb161816511695169e169e16a800000ed80ed80fd210cc11c6132414e615e01621163e164816441644165100000aa70aa70c310dfe0f6b10d712ae1381141b146814ab14db1501158800000708070808fc0ad70ca40ed810cc119411f8125c12c0134813881438000005bd05bd079d096a0b110d510f140ff11051109e11071141116711db00000514051406c0085d09dd0c1c0dba0e740ed80f3c0f6e0f6e0f6e0fae000004930493061d079408e40aea0c570d0e0d5a0da70dce0dd70de10dfe000004500450058406a407e009c40ae10b670bbd0c010c3a0c310c3a0c4400000420042004fd05f70708087009a40a210a640a9d0aba0aba0aba0ab1000003e803e8040d044704930537060a06c0070d0747075a075a075a075a 
# Oil_temp table (len 36)
0xdad3 000000fd00000000003c00000064002800fd00c100ff00ff00ff00ff00ff00ff00ff00ff

0xf110 44532d463145542d3130423934342d414100000000000000 # DS-F1ET-10B944-AA - ???
0xf111 463145542d3134473036322d414100000000000000000000 # F1ET-14G062-AA - ? (ECU Core Assembly Number)
0xf113 463145542d3130423934342d424100000000000000000000 # F1ET-10B944-BA - HW (ECU Delivery Assembly Number)
0xf124 463145542d3134473035372d414100000000000000000000 # F1ET-14G057-AA - SERVICE NVM DATA (ECU Calibration Data #1 Number)
0xf159 022106
0xf15b 012602
0xf15c 050400
0xf15e 030417
0xf15f 08070001010084010000
0xf160 050726
0xf161 021200
0xf162 05
0xf163 03
0xf166 11020415
0xf170 <empty>
0xf17b <empty>
0xf180 01434d35542d3134473035382d414100000000000000000000 # (0x01)CM5T-14G058-AA - ???
0xf188 463145542d3134473035362d414100000000000000000000   # F1ET-14G056-AA - ROM DATA (Vehicle Manufacturer ECU Software Number)
0xf18c 52434241303138323430000000000000                   # RCBA018240 - ??? (S/N) 

0xfd29 ce8b #backlight CH 17*
0xfd30 ce8b #backlight CH 16*
0xfd31 83e4 #backlight CH 18*
```

## Gauges values calculation algorithm

Algorithms for calculating gauge values[1] ​​based on data from HS-CAN,
are presented using C code in the file `example.c`.  
These algorithms was obtained by reverse engineering of the MCU firmware.  

[1] Gauge values are measured in internal units, **not in the values ​​​​that are
signed under the arrows**.  

I.e. for turbo PSI `0xCE` value reflects 30 PSI (max right position) on `F1ET-10B944-BA` cluster.  
The same is for oil temperature: `12c0` is lowest arrow position and `0x39d0` is highest position.  
**Only oil PSI value reflects arrow position in percents.**


Since the MCU architecture is PPC32 (big endian), in order for the code
to work correctly on x86 (LE), it is suggested to use a docker container for PPC.

Build `example.c`:
```shell
s390x-linux-gnu-gcc-14 example.c -o example.exe -static -lm -lc -lgcc -lc
```

Run docker container for PPC:
```shell
docker run --rm --privileged multiarch/qemu-user-static:register --reset
docker run -v ./:/test -it multiarch/ubuntu-core:s390x-focal /bin/bash
```

Run `example.exe`:
```shell
# /test/example.exe
```

As a result the table of all possible internal values in dependency
of CAN data will be printed.
