### Sharp MZ-700 SRAM/ROM Board ###

This is a paged SRAM/ROM board designed for the Sharp MZ-700, and it requires the [Sharp MZ-700 Backplane expansion board](https://github.com/mz-700/mz700-backplane).

There are 3 versions of this board, each functionning similarly, with the lastest version offering additional features detailed in the branch`readme.md` file. 

The 3 branches are:

- `128K`: This branch. A 8KB ROM + 128KB SRAM board.
- [`512K`](https://github.com/mz-700/mz700-sram-rom/tree/512K): Same board with 512KB SRAM
- [`dual512K`](https://github.com/mz-700/mz700-sram-rom/tree/dual-512K): Same board with 512KB Flash ROM+512 KB SRAM.

The dual512K board is a bit different. There is a specific `readme.md` in the branch page that describes the differences. 

#### How does it work? ####

The Sharp MZ-700 being a 8bit computer, it cannot access more than 64KB of memory. To access this extra memory, it will need to be paged into windows inside its memory map.  
Let's take a look at the 700 memory map:

![MZ-700 Memory Map](https://original.sharpmz.org/mz-700/images/poweron.gif)  
<sup><sub>From https://original.sharpmz.org</sub></sup>

As we can see, the Sharp MZ-700 has 64KB or DRAM + 2KB of ROM + 4KB of VRAM + some addresses reserved for I/Os.  
To get the full picture of the thing, please read the excellent description Karl-Heinz Mau made on his [site](https://original.sharpmz.org/mz-700/coremain.htm).  

The SRAM/ROM will be mapped at the top of the 700 memory map. From `$E100` to `$FFFF`. Why not from `$E000`? As you can see on this image, the addresses between `$E000` and `$E009` are reserved for I/Os. Actually, the addresses between `$E000` and `$E00F` are reserved for the I/Os. As I am not sure if any other device uses some addresses after `$E00F`. I prefered starting the mapping at `$E100`. We will lose 256 bytes on the first bank, but I guess it is OK.

Talking about banks, the SRAM/ROM are mapped onto 4 banks of 2KB (minus 256 bytes for the 1st one). This means that the SRAM is divided into blocks of 2KB (64 blocks for the 128K version and 256 blocks for the 512K version) and we can choose which of these blocks will be mapped to each of these banks. 

#### The bank addresses are: ####

|Bank #|Start Address|End Address|
|:----:|-------------|-----------|
|0|$E100|$E7FF|
|1|$E800|$EFFF|
|2|$F000|$F7FF|
|3|$F800|$FFFF|

#### The Registers ####

Use the DIP switch on the left of the board to select the base address of the registers. I will call it `BaseAddr`. All the registers are write-only.

|Register|IO Port|Description|
|---|--------|-----------|
|Bank0_block|`BaseAddr`|# of the 2KB SRAM memory block that will be mapped to bank #0|
|Bank1_block|`BaseAddr`+1| # of the 2KB SRAM memory block that will be mapped to bank #1|
|Bank2_block|`BaseAddr`+2| # of the 2KB SRAM memory block that will be mapped to bank #2|
|Bank3_block|`BaseAddr`+3| # of the 2KB SRAM memory block that will be mapped to bank #3|
|Banks_selection|`BaseAddr`+4| 4 bits to choose if the bank will be mapped to ROM (0) or SRAM (1) - 0 by default|
|Board_disable|`BaseAddr`+5| 0 or 1: disable the board altogether - 0 by default (enabled)|

#### Example ####

```
BaseAddr        EQU $A0    ; <- value set by the DIP switches
Bank0_block     EQU BaseAddr
Bank1_block     EQU BaseAddr+1
Bank2_block     EQU BaseAddr+2
Bank3_block     EQU BaseAddr+3
Banks_selection EQU BaseAddr+4
Board_disable   EQU BaseAddr+5

LD  A,10                ; SRAM block #10 ( from SRAM $2800 (10*2KB) to $29FF (10*2KB + 2KB) )
OUT (Bank2_block),A     ; The addresses $F000->$F7FF will point to SRAM block #10
LD  A,%0010             ; 
OUT (Banks_selection),A ; The block #2 will point to SRAM, the others to ROM
LD  A,1
OUT (Board_disable),A   ; The board is disabled
LD  A,0
OUT (Board_disable),A   ; The board is enabled
```

#### When is the board disabled? ####

The board can be disabled with the `Board_disable` register. There are other cases when the board is disabled automatically. As explained in Karl-Heinz's page, the MZ-700 memory layout can change by using the I/O ports $E0 to $E6. This is a table copied from the page. It shows how the memory layout changes by using these ports:

|port#|	$0000 - $0FFF |	$D000 - $FFFF|
| ----|-|-|
|$E0 |	RAM |	no action|
|$E1| 	no action |	RAM|
|$E2 |	ROM| 	no action|
|$E3 |	no action |	V-RAM and I/O ports|
|$E4 |	ROM| 	V-RAM and I/O ports|
|$E5 |	no action |	locked|
|$E6 |	no action |	unlock locked area by $E5 |

As we can see, after `OUT ($E1),A`, the top of the memory is used by the internal DRAM. The board is then disabled to avoid any conflict. The board is then automatically enabled when a `OUT ($E3),A` or `OUT ($E4),A` occurs. The board is also disabled after `OUT ($E5),A`. This locks the top of the memory, meaning that any access to the VRAM or the I/O ports won't work until unlocked (with $E6 or $E4).  
Technically, there is no conflict with the board when the upper memory is locked but it then requires some memory switching each time we want to access to the video or the i/o ports. If this is acceptable, it is easy to update the CPLD code to comment out this test and leave the board active even when the upper memory is locked.

The LED is ON when the board is enabled. OFF when disabled.

### Building the board ###

The board uses an `ATF1502AS` CPLD. This chip requires a specific programmer: The ATDH1150USB. Unfortunately, it is not cheap ($90). 

Here is the BOM:

|Ref|Name|Description|Seller|Seller Reference|Comments|
|---|---|---|---|--|--|
|U1|ATF1502AS|CPLD|[Mouser](https://www.mouser.com/ProductDetail/556-AF1502AS10JU44)|556-AF1502AS10JU44 |  |
|U1|Socket|PLCC44 Socket|[Mouser](https://www.mouser.com/ProductDetail/737-PLCC-44-AT)|737-PLCC-44-AT|
|U2|74HCT688|8 bit comparator|[Mouser](https://www.mouser.com/ProductDetail/595-CD74HCT688EE4)|595-CD74HCT688EE4 |
|U3-U4|74LS670|4x4 Register|[Mouser](https://www.mouser.com/ProductDetail/595-SN74LS670N)|595-SN74LS670N |LS because the HCT is really expensive|
|U5|AS6C1008|128KB SRAM|[Mouser](https://www.mouser.com/ProductDetail/913-AS6C1008-55PCN)|913-AS6C1008-55PCN |
|U6|28C64|8KB EEPROM|[Mouser](https://www.mouser.com/ProductDetail/556-AT28C64B15PU)|556-AT28C64B15PU|
|C1-C9|Capacitor|0.1uF 50V +/-10% 5mm Decoupling capacitor|[Mouser](https://www.mouser.com/ProductDetail/594-K104K15X7RF53H5G)|594-K104K15X7RF53H5G |
|R1|Resistor|300 ohm|[Mouser](https://www.mouser.com/ProductDetail/594-MBE04140C3300FC1)|594-MBE04140C3300FC1 |
|D1|LED|5mm LED|[Mouser](https://www.mouser.com/ProductDetail/%20593-LTH5MM12VFR4100)|593-LTH5MM12VFR4100 |
|J1|connector|Shrouded 2x5 male connector|[Mouser](https://www.mouser.com/ProductDetail/200-TST10501GD)|200-TST10501GD |
|J2|connector|1x34 angled 2.54mm pin header|[Mouser](https://www.mouser.com/ProductDetail/649-78938-434HLF)|649-78938-434HLF |
|RN1|Resistor|5x10K Resistor Network|[Mouser](https://www.mouser.com/ProductDetail/652-4606X-1LF-10K)|652-4606X-1LF-10K 
|SW1|Switch|5x DIP Switch|[Mouser](https://www.mouser.com/ProductDetail/774-1945MST)|774-1945MST
|||CPDL Programmer|[Mouser](https://www.mouser.com/ProductDetail/556-ATDH1150USB)|556-ATDH1150USB|



