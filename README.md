# voamca
VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version

Limited warranty: "VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version" and documentation are "as is" without any warranty. The licensee assumes the entire risk as to the quality and performance of the software. In no event shall "VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version" or anyone else who has been involved in the creation, development, production, or delivery of this software be liable for any direct, incidental or consequential damages, such as, but not limited to, loss of anticipated profits, benefits, use, or data resulting from the use of this software, or arising out of any breach of warranty.

Based on ideas from casainho and hurzhurz found here:
https://endless-sphere.com/forums/viewtopic.php?f=28&t=94351&hilit=TSDZ2

There's an Android app called "ebmDisplay VOAMCA", fully compatible with this firmware:
http://www.wptm.hu/ebmdisplay_voamca/
It shows up Battery voltage, current, watt and pedal cadence. And it is able to set assist between level 1-3 firmly. Works via Bluetooth and/or USB Serial connected to display cable.

Live demo:
https://www.youtube.com/watch?v=K6m0NgTeFAM&t=337s

It is based on factory default firmware. And it is aimed to be compatible with displays used with TSDZ2.
The new features can be reached with alternative displays programmed to the changed UART messages between motor controller and display.
There are changes compared to original factory default firmware:
```
-values in the communication UART message to display (Torque value initial and actual are replaced by battery voltage and current. The error code is replaced by pedal cadence.)
-one value in the communication UART message to motor (The 3rd value which is unused by the firmware is replaced by firm assist level value. It should not be higher then 50 or 0x32.)
```

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


Part firm assist level:
-----------------------
Original - puts decimal 10 to assist level 0.5 (hidden) value int.:
```
0x9b99:	 a6 0a	ld A, #$0a
```

New - puts register $80 as assist level value. Register $80 is the 3rd value in the communication UART message to motor controller:
```
0x9b99:	 b6 80	ld A, $80
```

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

New UART to motor:
--------------------
```
1	0x59	Start-Byte
2	0x40	Motor control flags
3	0x28	Firm assist level value (max. 0x32)
4	0x1C	Wheel size
5	0x00	unknown, probably unused by LCD
6	0x1B	Maximum speed
7	0xD0	Checksum
```

Summary of changes in machine code (only 9 bytes actually):
-----------------------------------------------------------
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
@@ -218,7 +218,7 @@
 :209B20008A3F123FA8B622B7DC3FE081B620B7DC3FE0B1D924153DD62605A600B7D6813AE1
 :209B4000D63DD727023AD781CC9BD9B0D9720E008A04A10025E0B1DB27082404B7DB20004E
 :209B60003CDB5F720800962D7200008C32720A0096197202008C0AA664B7DDA65AB7EC206C
-:209B800026A65AB7DDA632B7EC201CA65AB7DDA619B7EC2012A65AB7DDA60AB7EC2008A677
+:209B800026A65AB7DDA632B7EC201CA65AB7DDA619B7EC2012A65AB7DDB680B7EC2008A6F1
 :209BA0005AB7DDA60CB7ECB6DBA1022400B6EB97B6DB42A642629FB1D824155F97B69D42C9
 :209BC000B6D8629FAB002508B1F52404B7D62004B6F5B7D6A101250181A601B7D681815F89
 :209BE000B63F97B63E95A66462A60A62A300FF2503A6FF819F81A640C7500EA602C75013E5
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
