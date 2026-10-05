Standard Horizon GX2000 wiring
==============================

There are 5 cables coming out of the back: from left to right:
-  white/shield (4 Ohm PA/horn Speaker)
-  red/shield (4 Ohm external audio speaker)
-  black (power-)
-  red (power+)
and the last on the right is a 6 wire NMEA0183 cable containing: 
brown, yellow, white, gray, blue, green.

## NMEA0183 cable wires

NMEA cable contains 3 pairs of NMEA0183 +/- lines:   
-  NMEA IN: for getting GPS data @4800 only
-  NMEA OUT: for emitting DSC and DSE call status
-  NMEA HightSpeed IN: for getting both AIS and GPS data


| Color  | Diagram label  | Description                  | Sentences                                      |
|--------|----------------|------------------------------|------------------------------------------------|
| Blue   | NMEA IN (+)    | NMEA GPS Input (+)           | *No use* (@4800: GGA, GLL, GNS, RMC, GSA, GSV) |
| Green  | NMEA IN (–)    | NMEA GPS Input (–)           | *No use* (@4800: GGA, GLL, GNS, RMC, GSA, GSV) |
| Gray   | NMEA OUT (+)   | NMEA DSC Output (+)          | DSC, DSE (@4800 & @38400)                      |
| Brown  | NMEA OUT (–)   | NMEA DSC Output (–)          | DSC, DSE (@4800 & @38400)                      |
| Yellow | NMEA-HS IN (+) | NMEA-HS (AIS Data) Input (+) | VDM, GGA, GLL, GNS, RMC, GSA, GSV (@38400)     |
| White  | NMEA-HS IN (–) | NMEA-HS (AIS Data) Input (–) | VDM, GGA, GLL, GNS, RMC, GSA, GSV (@38400)     |
 

## VDM sentence

VDM = VHF Data-link Message. It's the NMEA 0183 sentence type that carries AIS data over a serial port. 

 
| | VDM | VDO |
|---|---|---|
| Full sentence | `!AIVDM` | `!AIVDO` |
| Meaning | Data **received** from other vessels via VHF AIS | **Own** vessel data |
| Talker ID | `AI` (AIS) | `AI` |


Why it exists: AIS transmits up to ~4,500 messages/minute in 6-bit binary (ITU-R M.1371). That's far too much for normal parametric NMEA sentences. So the entire binary AIS payload is stuffed into a single "encapsulated data field" — that's what VDM is. The ! prefix (instead of $) marks it as an encapsulation sentence, a special NMEA 0183 mechanism for exactly this purpose. 

Structure:

~~~
!AIVDM,1,1,,A,177l?m9000:Pk<ikh0ISd00R;,0*30
~~~


| Field | Meaning |
|-------|---------|
| `1` | Total number of sentences (fragments) |
| `1` | This sentence number |
| (empty) | Sequential message ID |
| `A` | AIS channel (A or B) |
| `177l?m9000...` | 6-bit encoded AIS payload |
| `0` | Fill bits |
| `*30` | Checksum | 


The 6-bit payload decodes into ITU-R M.1371 message types 1–27 (position reports, static data, safety messages, etc.). That's why you need 38,400 baud (NMEA 0183-HS) — at 4,800 baud the throughput isn't sufficient for a busy AIS channel.