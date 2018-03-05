# FT8Call Design Document

FT8 has taken over the airwaves as _the_ digital communication mode for making QSOs over HF/VHF/UHF. The mode has been widely popular as the latest mode offered in the WSJT-X application. It stands on the shoulders of JT65, JT9, and WSPR modes for weak signal communication, but much faster.

While FT8 is an incredibly robust weak signal mode, it only offers a minimal QSO operation framework, designed heavily to take advantage of short band openings on VHF/UHF. However, many operators are using these weak signal properties successfully on the HF bands. 

The idea here is to take the robustness of FT8 mode and layer on a messaging and network protocol for weak signal _communication_.  

## Goals 

1. Create a network of manned and automated stations communicating over FT8 mode
2. Offer intutive and short commands for passing messages within the network
3. Use efficient message packing to send vital information over the airwaves in a minimal amount of time
4. Allow keyboard to keyboard chat between stations with minimal overhead
5. Optionally allow reliable transmission of messages with automatic retransmission of failed message frames.

## Channel Allocation

Modes like FSQ and HFAPRS allocate a single channel for their communication. This is useful for keeping the bandwidth of transmissios in a reasonable range. However, stations within a single channel may cause each other interference when transmitting over to of each other.

Thankfully, FT8 mode transmissions are sent in a 50Hz bandwidth. What we can do is leverage this small bandwidth to allot several channels for multiple, parallel transmissions in-network. We can leverage the FT8 multi-decoder to allow for synchronized transmissions for multiple stations without overlapping / competing signals. 

How? We'll use 1500Hz as the center of the passband for our frequency, then use 50Hz channels above and below the center frequency spaced 10Hz apart 

```
|                         1500Hz                         |
|                           |                            | 
| [ 1355Hz ] [ 1415Hz ] [ 1475Hz ] [ 15685z ] [ 1640Hz ] |
|                                                        |
|                                                        |
```

Each station will randomly choose one of these slots as their default (or maybe they are assigned one based on a hash of their callsign?) Or maybe they choose even or odd and can synchronize that way. Each station sounding could include this information about channel usage. 

We could also set channels lower than 1500Hz  for message sending (directed, sounding, etc) and channels higher could be for replies. Or, replies could just stick to the channel on which the message was received. 

Each station would maintain a preferred list of channels for each station heard (i.e., which channel the station heard them on). 

Up to 25 channels could fit in a 1500Hz segment of the passband (1500Hz / (50+10Hz)). Each new station sounding would be trying to fill in the gaps to avoid broadcasting over top of each other. 

Alternatively, we could just rely on the operator to select a free channel in the passband.

One neat thing about this (mostly related to the FT8 modes fantastic decoder) is that multiple messages could be delievered simultaneously from multiple operators without interference.

## Messages

There will be two primary types of messages, each with sub-types: 

1. Sounding / beacon messages (station to all)

    * Ping
    * Ping-Req

2. Directed messages (station to station)

    * Command Requests
    * Acks (automated)
    * Free Text
    * Relay Free Text (automated)

Sounding messages would be used to announce a station to all listeners. Directed messages would be used to send commands or messages back and forth between stations. 

### Sounding

We could use the SWIM protocol for managing the network membership list. Basically, an efficient gossip mechanism to announce joins to the network. 

We wouldn't need to worry about dead links for the most part unless we wanted to clean up the heard list over time. Displaying time since last heard would be a nice to have. 

We could piggyback on sounding for disseminating any known membership changes. Sounding could occur on a random channel in the passband at a random minute offset if network membership changed. 

Receiving sounding with membership information could allow us to include 2-hop stations in the "heard" list.

### Directed 

Directed messages would be used to send information across the network. This could be commands to other stations, messages to other stations, or relays through other stations. 

### Structure

FT8 transmissions carry 75-bits of information. This information is transmitted redundantly using forward error correction. Normal FT8 transmissions are 72-bits long with 3-bits left unused. FT8 free text transmissions are 13 characters long at 5 bits per character, for a total of 65 bits.

We know that the EME paper [1] describes an efficient way to encode all known amateur radio callsigns in 28-bits of information. This is what JT65 [2] (and FT8) uses for their minimal QSO message packing. With this in mind, we can develop a simple command message protocol where most non-freetext messages are passed with only one 15-second FT8 transmission.

#### Base Message

```
+----------+------------+--------------+
| Head (1) | Type (1-2) | Data (72/73) |
+----------+------------+--------------+
| 0        | 00         | 0000....0000 |
+----------+------------+--------------+
```

**Head:** Identify if there are any more frames to the message.

* `|0|` = This is the last frame of the message
* `|1|` = There is at least one more frame to the message

**Type:** Identify the type of message. Type is a variable bit flag and can be 1 or 2 bits long. 

* `|1|`  = Type 1 (Directed Data)
* `|01|` = Type 2 (Sounding)
* `|00|` = Type 3 (Directed Header)

**Data:** The data in the message (depending on the message type)

#### Sounding Message

```
+----------+----------+---------------+-----------+
| Head (1) | Type (2) | Callsign (28) | Data (44) |
+----------+----------+---------------+-----------+
| 0        | 01       | 0000....0000  | 000...000 |
+----------+----------+---------------+-----------+
```

Data can encode any information within 44-bits

Likely candidates: 

* Sounding Type (3 bits)
* 8 text characters (40 bits)
* Another Callsign (28 bits)
    * Call command (3 bits)
        * REQ, SEEN
* Grid (15 bits)
  * Or a high precision grid locator (30 bits) [6]
* Signal Report (4 bits)
	* 0-12 = -2dB * bits = SNR
		* 0001 = -2dB * 1 = -2dB
		* 0010 = -2dB * 2 = -4dB
		* 1100 = -2dB * 12 = -24dB
	* 13-15 = 3dB * (16 - bits)
		* 1111 = 15 = 3*(16-15) = +3dB
		* 1110 = 14 = 3*(16-14) = +6dB
		* 1101 = 13 = 3*(16-13) = +9dB
* Power (3 bits) 
    0. 0-500mW
    1. 1W
    2. 2W
    3. 5W
    4. 10W
    5. 50W
    6. 100W
    7. 1KW
 
Sounding messages should include a sounding command when another callsign is sent along to identify if the sounding is a request or is a notification.


#### Directed Message

Message stream (Frame 1): 
```
+----------+----------+---------------+------------------+
| Head (1) | Type (2) | Callsign (28) | To Callsign (28) | 
+----------+----------+---------------+------------------+
| 0        | 00       | 0000.....0000 | 0000........0000 |
+----------+----------+---------------+------------------+
+-----------------+-------------+----------+
| Frame Count (4) | Command (4) | Args (8) |
+-----------------+-------------+----------+
| 0000            | 0000        | 0000     |
+-----------------+-------------+----------+
```

**Pad:** Identifies the number of pad bits at the end of the message (0-7 bits)

**Frame Count:** Number of frames to follow

**Command:** Directed command

**Args:** Arguments to the directed command, likely 8 bit int


---

Message stream continuations (Frames 2 through n): 
```
+----------+----------+---------+-----------+
| Head (1) | Type (1) | Pad (3) | Data (70) | 
+----------+----------+---------+-----------+
| 1        | 1        | 000     | 000...000 |
+----------+----------+---------+-----------+
```

**Pad:** Identifies the number of pad bits at the end of the message (0-7 bits)

**Data:** Huffman encoded varicode data

**CRC:** The last frame of the message should include a CRC of the message to ensure it was correctly assembled.



### Message Examples

* Sounding
    * ping - here is my station info
        * ping [callsign] seen - i saw this callsign
        * ping [callsign] req - has anyone heard this callsign?
    * ack [callsign] [bitflag] - i've received your messages (and may be missing some in the bit flag)
* Directed
    * allcall [message] - broadcasting this message to all stations in network
    * cqcqcq [message] - broadcasting this message to those with cq enabled
    * [callsign]:? - what is my signal report
    * [callsign]:![message] - relay this message
    * [callsign]:@ - what is your station qth (location)?
    * [callsign]:& - what is your station qtc (message)?
    * [callsign]:$ - what stations have you heard?
    * [callsign]:% - what is your station information?



Sounding Exmamples: 

```
   1  2   28      3     15    15    3   
->[0][01][KN4CRD][PING][EM73][TU00][5W] = 67
```

```
   1  2   28      3        28     
->[0][01][KN4CRD][PINGREQ][OH8STN] = 62
   1  2   28      3     28      4
<-[0][01][VA3OSO][SEEN][OH8STN][-15dB] = 66
```

```
   1  2   28      3     28      4
->[0][01][KN4CRD][SEEN][OH8STN][-15dB] = 66
```


Direct Examples: 

Direct Free Text: 
```
   1  2   28      28      4        4    8        
->[1][00][KN4CRD][KN4CRD][1 FRAME][MSG][00000000] = 75 
   1  1  3    63
->[0][1][111][how are you??] = 75
```

Direct Command (report): 
```
   1  1   28      28      4         4      8
->[0][00][KN4CRD][OH8STN][0 FRAMES][CMD ?][00000000] = 75
   1  2   28      3     28      4
<-[0][01][OH8STN][SEEN][KN4CRD][-15dB] = 66
```

Direct Command (heard list)
```
   1  1  28      28      4         4      8
->[0][1][KN4CRD][OH8STN][3 FRAMES][CMD $][000000000] = 75
   1  2   28      3     28      4
<-[1][01][OH8STN][SEEN][KN4CRD][-16dB] = 66
<-[1][01][OH8STN][SEEN][ K1JT ][-12dB] = 66
<-[0][01][OH8STN][SEEN][ K4YY ][-02dB] = 66
```

Direct Command (station info): 
```
   1  2   28      28      4         4      8
->[0][00][KN4CRD][OH8STN][0 FRAMES][CMD %][00000000] = 75
   1  2   28      4     15    15    3
<-[0][01][OH8STN][PING][EM73][TU00][5W] = 67
```


### SRARQ - selective repeat - automatic repeat requests

When used as the protocol for the delivery of subdivided messages SRARQ works somewhat differently. In non-continuous channels where messages may be variable in length, standard ARQ or Hybrid ARQ protocols may treat the message as a single unit. Alternately selective retransmission may be employed in conjunction with the basic ARQ mechanism where the message is first subdivided into sub-blocks (typically of fixed length) in a process called packet segmentation. The original variable length message is thus represented as a concatenation of a variable number of sub-blocks. While in standard ARQ the message as a whole is either acknowledged (ACKed) or negatively acknowledged (NAKed), in ARQ with selective transmission the ACK response would additionally carry a bit flag indicating the identity of each sub-block successfully received. In ARQ with selective retransmission of sub-divided messages each retransmission diminishes in length, needing to only contain the sub-blocks that were linked. [5]

## References

[1]: EME 2000 - http://www.ka9q.net/papers/eme-2000.ps.gz
[2]: The JT65 Communications Protocol - http://www.arrl.org/files/file/18JT65.pdf
[3]: SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol - https://pdfs.semanticscholar.org/8712/3307869ac84fc16122043a4a313604bd948f.pdf
[4]: WSPR Coding Process -  http://www.g4jnt.com/Coding/WSPR_Coding_Process.pdf
[5]: ARQ - https://en.m.wikipedia.org/wiki/Selective_Repeat_ARQ
[6]: Maidenhead Grid - http://www.jidanni.org/geo/maidenhead/index.html
