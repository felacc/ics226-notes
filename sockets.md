# Sockets (in Python)
Sockets are data structures that abstract communication endpoints.
All POSIX-compliant systems support sockets (POSIX = Portable Operating System Interface).
To recieve messages sent to a specific port (1-65535), a process must first "bind" to a socket.

**Note**: the first 1024 ports require root privieleges in order to bind to them. These ports are reserved for specific services.
## UDP Overview
UDP is fast but unreliable transport layer protocol.

### Sample UDP Server
```python
#!/usr/bin/python3.11
from socket import socket, AF_INET, SOCK_DGRAM

BUF_SIZE = 1024
HOST = '127.0.0.1' # '' would cause the server to listen on all interfaces
PORT = 65432

with socket(AF_INET, SOCK_DGRAM) as sock: # UDP Socket
    sock.bind((HOST, PORT)) # Claim messages sent to port "PORT"
    data, address = sock.recvfrom(BUF_SIZE) # Receive up to "BUF_SIZE" bytes
    text = data.decode('utf-8') # Convert from binary representation
    source_ip, source_port = address # Extract source IP and source port
    print(f'IP: {source_ip} Port: {source_port} Message: {text}')
```

### Sample UDP Client
```python
#!/usr/bin/python3.11
from socket import socket, AF_INET, SOCK_DGRAM
from sys import argv

BUF_SIZE = 1024
HOST = '127.0.0.1'
PORT = 65432

if len(argv) != 2:
    print(argv[0] + ' <message>')
    exit()
    
with socket(AF_INET, SOCK_DGRAM) as sock: # UDP socket
    data = argv[1].encode('utf-8') # Convert command line arg to binary
    sock.sendto(data, (HOST, PORT)) # Send message to server
```

#### Test the code
- Enter `./client.py "This is a test"`
- Server prints `IP: 127.0.0.1 Port: 57937 Message: This is a test`
- The source port number will vary, it is assigned dynamically

### Setting Up a UDP Server
- Create a socket data structure: `with socket(AF_INET, SOCK_DGRAM) as sock`
    - `SOCK_DGRAM` tells socket to use UDP 
- Claim (bind to) a port: `sock.bind((HOST, PORT))`
    - Note that this takes a tuple
    - Tells the operating system that the calling program want to recieve UDP message to the given host and port
    - To recieve messages from the network, HOST must either be blank or set to the IP address of the network card
    - To recieve messages from local host, use localhost or 127.0.0.1\

### Setting Up a UDP Client
- Create a socket data structure: `with socket(AF_INET, SOCK_DGRAM) as sock`
    - `SOCK_DGRAM` tells socket to use UDP 

### Recieving via UDP
- To recieve a network message: `recvfrom(BUF_SIZE)`
    - `data, address = sock.recvfrom(BUF_SIZE)`
- BUF_SIZE is the maximum acceptible size, it is not a guarantee, it could return less bytes
    - If exactly n bytes are needed, recvfrom will have to be called repeatedly
- The address returns a tuple with the sender's IP and port
- Recieved messages are encoded in binary format
    - Before processing message must be decoded `text = data.decode('utf-8')`
        - utf-8 is the default arg for decode()

### Sending via UDP
- To send a message: `sendto(data, (host_ip, host_port))`
- Transmitted data must be encoded: `sock.sendto(text.encode(), (source_ip, source_port))`
- There is a chance `sendto` won't send all bytes (bc it's UDP)
- `sendto` returns the actual number of bytes sent, so that you can retransmit the remaining message if necessary
- UDP messages can be lost or duplicated it is up to the application to deal with these issues

## TCP
TCP is reliable but slower transport layer protocol.

### Sample TCP Server
```python
from socket import socket, AF_INET, SOCK_STREAM, SOL_SOCKET, SO_REUSEADDR

BUF_SIZE = 1024
HOST = ''
PORT = 65432

with socket(AF_INET, SOCK_STREAM) as sock: # TCP socket
    sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1) # Details later
    sock.bind((HOST, PORT)) # Claim messages sent to port "PORT"
    sock.listen(1) # Server only supports a single 3-way handshake at a time
    print('Server:', sock.getsockname()) # Server IP and port
    while True:
        sc, _ = sock.accept() # Wait until a connection is established
        with sc:
            print('Client:', sc.getpeername()) # Client IP and port
            data = sc.recv(BUF_SIZE) # recvfrom not needed since address known
            decoded_data = data.decode()
            print(decoded_data)
            sc.sendall(data) # Client IP and port implicit due to accept call
```

### Sample TCP Client
```python
#!/usr/bin/python3.11
from socket import socket, AF_INET, SOCK_STREAM
from sys import argv

BUF_SIZE = 1024
HOST = '127.0.0.1'
PORT = 65432

if len(argv) != 2:
    print(argv[0] + ' <message>')
    exit()

with socket(AF_INET, SOCK_STREAM) as sock: # TCP socket
    sock.connect((HOST, PORT)) # Initiates 3-way handshake
    print('Client:', sock.getsockname()) # Client IP and port
    data = argv[1].encode('utf-8') # Convert command line arg to binary
    sock.sendall(data) # Server IP and port implicit due to connect call
    reply = sock.recv(BUF_SIZE) # recvfrom not needed since address known
    decoded_reply = reply.decode()
    print('Reply:', decoded_reply)
```

### Timeout Notes
- If an 'Address already in use' error occurs, the socket is still in use, possibly by an old version of the server.
- Normally terminating the server does not immediately close the socket, since TCP requires the use of timeouts (Two Army Problem).
- Terminating a server and restarting it can cause bind to fail because the system is keeping the socket until the timeout occurs.
- `SO_REUSEADDR` is used to make debugging easier by overriding the timeout feature.

### Sending and Recieving Notes
- `sendall` will send all data, but may split the message into several parts
- `recv(n)` will read up to `n` bytes, but could be less. It is up to application to check number of bytes recieved.

### Setting up a TCP Server
- Create a server socket data structure: `with socket(AF_INET, SOCK_STREAM) as sock:`
    - `SOCK_STREAM` tells the socket to use TCP
- Claim (bind to) a port: `sock.bind((HOST, PORT))`
- Set up a queue: `sock.listen(n)`
    - Tells operating system to allow up to `n` connections
    - Any additional clients attempting 3-way handshake will be ignored, until program calls the `accept()` function
- Accept a connection: `sc, _ = sock.accept()`
    - Returns a client socket and information about the connected socket in a tuple
    - Once a client socket is available to the program, it can be used to send and recieve messages to and from the client

### Setting up a TCP Client
- Create a socket data structure: `with socket(AF_INET, SOCK_STREAM) as sock:`
    - `SOCK_STREAM` tells the socket to use TCP
- Connect to server: `sock.connect((HOST, PORT))`

### Recieving via TCP
- To recieve a network message: `recv(BUF_SIZE)`
    - BUF_SIZE is maximum acceptible size, could be less
- `recv` only returns the data, since originating client is already known
- Data must be decoded: `data.decode('utf-8')`

### Sending via TCP
- Send a message: `sc.sendall(text.encode('utf-8'))`
    - Transmitted data msut be encoded
    - Transmissions are buffered, so sending `n` bytes with `sendall` does not mean `recv(n)` will recieve all those bytes
        - May have to call `recv` multiple times
        - call `data = data + recv(n - len(data))` repeatedly

#### Read Until Newline
If we know each message ends with a newline, we can call `data = recv(1)` until a newline is recieved...for now. In production, this approach would have to be optimized.

```python
BUF_SIZE = 1 # Not efficient

def get_line(current_socket: socket) -> bytes:
    buffer = b''
    while True:
        data = current_socket.recv(BUF_SIZE)
        if data == b'' or data == b'\n':
            return buffer
    buffer = buffer + data
```

## Data Encoding


