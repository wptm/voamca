# voamca
VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version - enabled for both 36v and 48v batteries.
This is kind of patch development to be compatible with Android app ebmDisplay: http://www.wptm.hu/ebmdisplay/ 

Limited warranty: "VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version" and documentation are "as is" without any warranty. The licensee assumes the entire risk as to the quality and performance of the software. In no event shall "VOAMCA (Voltage Ampere Cadence) TSDZ2 firmware version" or anyone else who has been involved in the creation, development, production, or delivery of this software be liable for any direct, incidental or consequential damages, such as, but not limited to, loss of anticipated profits, benefits, use, or data resulting from the use of this software, or arising out of any breach of warranty.

Based on ideas from casainho and hurzhurz found here:
https://endless-sphere.com/forums/viewtopic.php?f=28&t=94351&hilit=TSDZ2

It is based on factory default firmware. And it is aimed to be compatible with displays used with TSDZ2.
The new features can be reached with alternative displays programmed to the changed UART messages between motor controller and display.
There are changes compared to original factory default firmware:

-values in the communication UART message to display (Torque value initial and actual are replaced by battery voltage and current. The error code is replaced by pedal cadence.)

-two values in the communication UART message to motor (The 3rd value which is unused by the firmware is replaced by firm assist level value. It should not be higher then 50 or 0x32.) (And the 5th value which has an unknown functionality that is only activated if value 0xAA(170) is sent. So it is wise to avoid sending 0xAA(170) there.)

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

New - puts register $80 as assist level 0.5 (hidden) value. Register $80 is the 3rd value in the communication UART message to motor controller:
```
0x9b99:	 b6 80	ld A, $80
```

Part walk assist RPM:
-----------------------
Original - puts decimal 100 to walk assist RPM:
```
0xb873:	 a6 64	ld A, #$64
```

New - puts register $87 as walk assist RPM. Register $87 is the 5th value in the communication UART message to motor controller:
```
0xb873:	 b6 87	ld A, $87
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
5	0x00	Walk assist RPM. (max. 0xA9(169)). Should not be 0xAA(170) because this value activates unknown function.
6	0x1B	Maximum speed
7	0xD0	Checksum
```
