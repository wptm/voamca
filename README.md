# voamca
VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version

Limited warranty: "VOAMCA (Voltage Ampere Cadence) TSDZ2 36v firmware version" and documentation are "as is" without any warranty. The licensee assumes the entire risk as to the quality and performance of the software. In no event shall "VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version" or anyone else who has been involved in the creation, development, production, or delivery of this software be liable for any direct, incidental or consequential damages, such as, but not limited to, loss of anticipated profits, benefits, use, or data resulting from the use of this software, or arising out of any breach of warranty.

Based on ideas from casainho and hurzhurz found here:
https://endless-sphere.com/forums/viewtopic.php?f=28&t=94351&hilit=TSDZ2

There's an Android app called "ebmDisplay VOAMCA", fully compatible with this firmware:
http://www.wptm.hu/ebmdisplay_voamca/
It shows up Battery voltage, current, watt and pedal cadence. Works via Bluetooth and/or USB Serial connected to display cable.

Live demo:
https://www.youtube.com/watch?v=K6m0NgTeFAM&t=337s

It is based on 36v factory default firmware. And it is aimed to be compatible with displays used with TSDZ2.
The new features can be reached with alternative displays programmed to the changed UART message sent from motor controller.
Only the values in the communication UART message to display are changed compared to original firmware.
The torque value initial and actual are replaced with battery voltage and current. The error code is replaced with pedal cadence.

Part voltage and current:
-------------------------
Original - torque initial and actual:
```
0x82d7:	 b6 e1	ld A, $e1
0x82d9:	 b7 78	ld $78,A
0x82db:	 b6 20	ld A, $20
0x82dd:	 b7 79	ld $79,A
```

New - battery voltage and current:
```
0x82d7:	 b6 1F	ld A, $1F
0x82d9:	 b7 78	ld $78,A
0x82db:	 b6 1D	ld A, $1D
0x82dd:	 b7 79	ld $79,A
```

So, new VOAMCA version puts register $1F(battery voltage V=x/2.9) to $78. And it puts $1D(battery current A=x/5) to $79.
$78 and $79 are the 4th and 5th data bytes in the communication UART to the display. 
They originally contain the torque initial, and torque actual values. XH-18 does not show up these values, so it's still compatible with this display.


Part cadence:
-------------
Original - error code lookup:
```
0x9f0f:	 72 08 00 8d 03	btjt $8d, #4, $9f17  (offset=3)
```

New - cadence value to 7A:
```
0x9f0f:	 b6 04 b7 7A 81	"ld A, $04" | "ld $7A,A" | "ret"
```

Well, this is trickier. The original code looks up if error has occured. When not, then it clears with "clr $7a" as next instruction.
I filled the 5 bytes with two "ld" (load) and a "ret". So, it will put cadence from register $04(pedal cadence RPM=2x) to $7A and returns before "clr $7a" comes as next.
$7A is the 6th data byte in the communication UART to the display.
This originally contains the error code (if any). XH-18 does not show up this value, so it's still compatible with this display.


New UART to display:
--------------------
```
1	0x43	Start-Byte
2	0x00	Battery level
3	0x01	Motor status flags
4	0x78	Battery voltage(V=x/2.9)
5	0x00	Battery current(A=x/5)
6	0x00	Pedal cadence(RPM=2x)
7	0x07	Speedsensor (LOW part of 16bit int)
8	0x07	Speedsensor (HIGH part of 16bit int)
9	0xF4	Checksum
```


Summary of changes in machine code:
-----------------------------------
```
--- Factory default: program_memory.hex
+++  VOAMCA version: voamca_program_memory.hex
@@ -20,7 +20,7 @@
 :2082600094CD9E94CD9E94CD9E94CD9E94CD9E94CD9E94CD9E94CD9E94CD9E94CDA2CD2017
 :20828000007215008FCD9C06CD8991CDAC33CD85BD72180097CDA38CCDA030CDA080CDA79C
 :2082A000C1CDA7FDCD9ECDCD9F0FCD82E0CD82F5CD82C97212008F7208008D077202008F2A
-:2082C000022000CD9E8ACC8291B67CA1062504A6152002A6BCB70DB6E1B778B620B7798156
+:2082C000022000CD9E8ACC8291B67CA1062504A6152002A6BCB70DB61FB778B61DB779811B
 :2082E000B61FB16D2406721D00902008721C00907218008D81A685B1FB2525A676B1FB2556
 :208300002CB6FBA12D2510A02D5F97D68337B1FC271B240C3AFC81A60AB1FC24113AFC810B
 :20832000B6FCA16424033CFC81A664B7FC81721800947218008D810103050608090A0C0D6F
@@ -246,7 +246,7 @@
 :209EA00048721D0089CD9E8A720D0089F83A003D0026EE81A664B700CD9E8ACD9EC33A0023
 :209EC00026F681AE14965A26FD813FB181720F008A08B6B04848B11D25F0B6B1A16425ECB5
 :209EE0003FB1B61FB1A22504B1A324073F76721000778172110077B0A3B776A11E2502A66D
-:209F00001E5F97B6AD42B6AE629FBBAFB776817208008D033F7A81720F008A0183720600C0
+:209F00001E5F97B6AD42B6AE629FBBAFB77681B604B77A813F7A81720F008A01837206005E
 :209F20008D70720F526D6D720A008F55720000891D720C008F36720000932F720A008D2A55
 :209F40002061B6FBA1852566721900942055B61FB1A2255AB6A4A1042404A604B7A43CA4D7
 :209F6000721100897219008F20392042B65AA155243C7204008A377219008A721D008F2010
```
