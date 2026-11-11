# TCP-IP
## NETWORK REVIEW
- Network layers
- Network headers
- Network protocol flow

## NETWORK LAYER REVIEW
### Physical Layer (1):
- Responsible for the physical connections between adjacent network devices
- Determines how data is represented and transmitted over cables in electromagnetic form

### Data Link Layer (2):
- Responsible for getting data from one device to another on the same network
- Breaks data up into individual FRAMES
- Handles error detection and correction as needed
- Works with MAC addresses

Note: Ethernet encompasses both phisical and data link layers

### Network Layer (3):
- Responsible for determining network ROUTES
- Moves PACKETS from source net to destination net
- Works with IP

### Transport Layer (4):
- Deals with sender and reciever issues: (De)multiplexing, packet ordering, flow control, etc.
- TCP and UDP
- TCP = reliable but slow
- UDP = unreliable but fast

## HEADERS
### IPV4 HEADERS
version, header length, dscp, ecn, header + data length, id, flags, offset, ttl, proto, header checksum, src ip, dst ip, opts 
Important components:
- DSCP = Differentiated Services Code Print - used to describe the payload (i.e. VOiP)
- ECN = Explicit Congestion Notification - used to alert destination of impending congestion, then used to alert source
- ID + Flags + Offset - used to enable/identify fragments of packets
- TTL = Time to Live - the per-router hop count a packet can make before being discarded
- Proto - identifies the payload type

### TCP HEADERS
src port, dst port, sequence number, ack number, header length, 0, flags, window size, chksum, urgent pointer, opts
Important components:
- Urgent Pointer - points to location of the urgent data in the payload section (if URG flag is set)

### UDP HEADERS
src port, dst port, header + data length, checksum (optional on IPv4)


## TCP CONNECTION - THE 3-WAY HANDSHAKE
client: SYN Flag, Sequence Number
server: SYN + ACK Flags, Sequence Number, Ack Number, RWND
client: ACK Flag, Sequence Number, ACK Number, RWND

Parts of connection establishment:
- ACK = acknowledgement
- RWND = Recieve window (used for flow control) - tells the other computer how much additional data the reciever is currently willing to accept
- SYN = synchronize sequence numbers

Rules:
- SYN and SYN/ACK segments do not carry any data
- An ACK without data does not consume a sequence number
- Full duplex requires two recieve windows

## SEQUENCE NUMBERS
- The initial sequence number is always random (note: wireshark obscures this)
- Subsequent numbers depend on number of bytes transmitted
    - assume first seq is 10001
    - if 1000 bytes are transmitted, the sequence number range will be 10001 to 11000
    - the next segment will start at 11001

### A NOTE ON SEQ AND ACK NUMBERS
- seq numbers are sent with data, each byte consumes 1 byte of data
- ack numbers are set to the last seq recieved from the other device/process + 1
    - this is to indicate that the next sequence number expected to be sent is the ack number

### IMPORTANT CONCEPTS:
- A TCP connection is established using a 3-WAY HANDSHAKE
- During TCP connection establishment, if the server receives a packet with the SYN flag set
  and a sequence number set to 18753, it must reply with: the SYN and ACK flags set,
  the sequence number to it's own randomly generated number and the acknowledgement number
  set to the last recieved sequence number + 1.


## TCP DATA TRANSFER
- The PSH (push) flag - tells destination that buffered data should be made available right away
  rather than delay for more data to accumulate
- The sequence number will always refer to the first byte in the segment
- The window size tells the reciever how much data the sender is willing to accept
- The ACK number tells the reciever which sequence number the sender is expected to recieve next
- Urgency can be emphasized with the URG flag; points to urgent data in the data stream and causes signal to be sent to listening application

### IMPORTANT CONCEPT:
If the TCP segment sequence number is set to 58293 and 100 bytes are set,
the acknowledgement number must be: 58393. The sequence numbers consumed will be:
58293-58392.

## TCP CONNECTION TERMINATION
client: FIN
server: FIN + ACK
client: ACK 

### CONNECTION TERMINATION NOTES
- FIN/FIN+ACK consumes 1 sequence number even if it contains no data
- Last ACK contains no data
- But what if more data must be sent? (e.g., a database query takes time to retrieve)
    The TCP Half Close
    - client: FIN
    - server: ACK
    - server: sends remaining data segments
    - client: acknowledges data segments recieved
    - server: FIN
    - client: ACK

## THE TWO ARMY PROBLEM
What happens if the final ACK in data transmission gets lost?
- How do we know that the left-hand side won't wait forever for a FIN from the right-hand side
- Do we ACK the ACK?
- We can never gurantee that the final ACK is recieved by the other side

### TCP ERROR HANDLING OF THE TWO ARMY PROBLEM
- To handle lost or damaged segments, timers are set by both sides
  to determine whether or not to retransmit data or ACKs

**Rule 1**: When there is data to be sent, the sender must include an ACK,
        with the acknowledgement number set to the next sequence number it
        expects to recieve from the reciever.
- Data, sequence number, and ack are all sent together
- Piggybacking an ACK

**Rule 2**: When the recipient has recieved a valid segment from the sender,
        and the recipient has no data to send in return, 
        it must set a timeout after which an ACK must be returned to the sender,
        with the acknowledgement number set to the next expected sequence number.
- 'The delayed ACK'

**Example**
1. Blue sends data to red, sequence number is 1000 and 100 bytes of data
2. Red recieves data, expects sequence number 1100 from blue, but red has no data to send back
3. Red starts an ack timer, and waits briefly to see if it has data to send
4. Timer expires, and red sends a standalone ack with 1100

**Rule 3**: When a second unacknowledged segment arrives,
        the recipient must return an ACK to the sender,
        with the acknowledgement number set to the next expected sequence number.
- Building on rule 2
- Blue sends two segments before Red's ACK timer expires
- Red doesn't keep waiting, it immediately sends an ACK

**Example**
1. Blue sends segment 1, seq = 1000 with 100 bytes
2. Red recieves segment 1 and starts timer, expecting 1100
3. Blue sends 1100 with 100 bytes
4. Red recieves segment 2, Red expects 1200 and sends ack with 1200 immediately

Imporant note: red immediately sends 1200

**Rule 4**: When an out-of-order sequence number arrives,
        the segment is buffered and an ACK is returned to the sender,
        with the acknowledgement number set to the next expected sequence number.
 - 'Hey I missed some data, please send it to my, I will store the other data in the meantime'

 **Rule 5**: When a missing segment arrives, the recipient returns an ACK to the sender, with the acknowledgement number set to the next expected sequence number.
- 'Alright I got the missing data, resume normal flow'

**Fast Retransmission Rule**: If a 3rd duplicate ACK is recieved, the subsequent segment is deemed lost, its retransmit timer is restarted, and the segment is retransmitted.
- This allows data to be sent much faster, as multiple packets can arrive and be buffered before data is retransmitted.


**REMEMBER**: Retransmit timers are set after data is sent, in case an ACK is lost, it will automatically resend the data.

**Rule 6**: When a duplicate segment is recieved, it is discarded,
but the recipient returns an ACK to the sender, with the acknowledgement
number set to the next expected sequence number.
- This is to account for if an ack is lost

## IMPORTANT INFO - Excercises
- If a valid TCP segment is recieved from a sender, the recipient must set a timeout and set the acknowledgement
number to the next expected sequence number, assuming the recipient has no data to send. 
    - Rule 2
    - ACK timer
- If a duplicate TCP segment is recieved from the sender, the recipient must discard the segment,
return an ACK to the sender with the acknowledgement number set to the next expected sequence number.
    - Lost ack
    - Rule 6
- Assuming the recieve buffer already contains bytes 1000-1200
and a TCP segment arrives with sequence number 1400 and length 100,
the recipient must buffer the data and send an ACK with the acknowledgement number set to 1201
    - out-of-order sequence number
    - Rule 4

## KEY SKILLS
- Explain TCP connection establishment
- data transmission
- error control
- flow control
- and connection termination




