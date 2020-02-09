## The OSI Burger

A friend sends me a message on Twitter and a few milliseconds later I receive the message. We go on sending each other messages but then I stop and wonder how the internet is able to send these messages seamlessly. After a little research, I learn about the OSI model.

OSI stands for Open System Interconnection. In simple terms, it is how two computers communicate with each other or how data is transferred from one computer to another. You might say to yourself “okay computers talk to each other that’s simple” it wasn’t always simple. When computers started being manufactured in a large scale, different vendors designed computers with different hardware and software architecture. A standard way of communication was needed. The OSI model was developed by the International Organization for Standardization (ISO) in 1984 to solve this problem.

Okay great! We know what OSI means now let's talk about why this article calls it a burger 🍔 but before that let’s understand the structure of a typical burger. A burger has two buns. One at the top and one at the bottom with layers in between them. The structure of the OSI model isn’t so different from a burger (at least to me). The two buns of the burger can be used to represent two separate computers and we’ll be exploring the layers in between these computers that facilitates smooth communication back and forth.

The OSI burger has 7 layers arranged from the highest level of abstraction to the lowest level of abstraction. A layer can receive messages from the layer above it, process or transform it and pass it to the layer below it. It’s also good to note that a layer can also receive a message from the layer below and send it to the layer above depending on the direction of the communication. The 7 layers are:

- The Application Layer
- The Presentation Layer
- The Session Layer
- The Transport Layer
- The Network Layer
- The Data Link Layer
- The Physical Layer

Our burger is becoming easier to visualize now that we know the layers that exist in between the buns. Now let’s open this burger up and understand what each layer is made of.

#### The Application Layer

Commonly used computer applications that use the internet or a network function on this layer of the OSI burger. It is the layer a computer user uses to send and receive messages over the internet. This layer uses protocols like [HTTP(S)](https://en.m.wikipedia.org/wiki/Hypertext_Transfer_Protocol), [FTP](https://en.m.wikipedia.org/wiki/File_Transfer_Protocol), [SMTP](https://en.m.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol), [TELNET](https://en.m.wikipedia.org/wiki/Telnet), etc. Examples of applications that help communicate on this layer are Web browsers like Google Chrome and Firefox, Mail Clients, File transfer clients and the list goes on.

#### The Presentation Layer

Messages from the application layer go to the presentation layer. These messages are usually in form of characters and numbers. The presentation layer converts these characters and numbers into machine readable binary format. The process of conversation from characters and numbers to binary format or vice versa by the presentation layer is called **Translation**. 
To speed up transmission of data, this layer reduces the amount of binary bits using **data compression**. The compression could be lossy or lossless depending on the nature of the data. 
The presentation layer goes one step further to ensure data security by encrypting outgoing data and decrypting incoming data. The [SSL protocol](https://www.digicert.com/ssl/) is used by this layer.

#### The Session Layer

This layer is responsible for setting up and terminating connection between parties on the internet. It works by using APIs, one of which is [NETBIOS](https://en.m.wikipedia.org/wiki/NetBIOS). Before a connection is secured with a server (another computer) on the internet, the server has to verify the identity of the client. This is usually done with a username and password. This process is called **Authentication**. The session layer on the server also needs to confirm that the client has the right to access a resource. This step is called **Authorization**. This layer takes on one more responsibility which is to keep track of where requested resources are sent to. This function is called *Session management*.

It is important to note that a web browser usually performs the functions of the application layer, presentation layer and session layer.

#### The Transport Layer

Data from the session layer moves to the transport layer. The transport layer is responsible for segmentation, flow control and error control. 
To handle segmentation, the layer breaks data into *segments*. Each segment is assigned a port number and sequence number to help the receiver identify which application sent the data and reconstruct the data from the segments. The transport layer is said to be also responsible for *flow control* because it controls the rate and efficiency of data flow to and from the computer. A server might have the capacity to send data at 200mps but the client machine can only receive data at 2mps. The transport layer negotiates with the server to reduce the transfer speed to 2mps. If the client’s capacity is larger than the current transfer speed the transport layer also negotiates a higher transfer speed from the server. Sometimes, data arrives with some part missing. The transport layer controls these errors by using a technique called Automatic Repeat Request to perform another transfer to fix the error. Data is validated using checksums. Protocols used by the transport layer are [Transmission Control Protocol (TCP)](https://en.m.wikipedia.org/wiki/Transmission_Control_Protocol) and [User Datagram Protocol (UDP)](https://en.m.wikipedia.org/wiki/User_Datagram_Protocol). TCP enables connection oriented transmission and provides feedback on the transfer state of the data and is used where delivery is necessary while UDP enables connection less transmission where the delivery is allowed to fail. UDP is usually used by streaming services, VOIP etc.

#### The Network Layer

Routers live on this layer of the burger. The network layer performs logical addressing, path determination and routing roles. Each computer on a network is identified with an IP address, the network layer attaches the IP addresses of the sender and receiver of a data segment to each segment sent by the transport layer to form a *Packet*. The IP addresses are added to help the data arrive at its intended destination. The process of transforming a segment to a Packet is called **logical addressing**. 
The network also handles routing. **Routing** is the method of moving data packets from source to destination using the addressing done during the logical addressing stage. 
Two computers can be connected in different number of ways. The network layer chooses the best possible path for information flow using **Path determination**. A couple of path determination algorithms can be used for this purpose.

#### The Data Link Layer

Packets from the network layer are sent to the data link layer. After the network layer adds the IP addresses of the sender and receiver to form a packet. The data link layer receives this packet and adds the [MAC addresses](https://en.m.wikipedia.org/wiki/MAC_address) of both computers to the packet to form a *Frame*. This transformation is called **Physical addressing**. The data link layer is embedded as software in the computer’s Network Interface Card (NIC). A MAC address is assigned to each card by its manufacturer. 
The data link is the means data is transferred from one computer to another via a local medium. A media in this case refers to physical links between two computers. This could be a copper cable, an optical cable or air (radio signals). The data link controls how data is received and placed on the media using techniques like Media Access Control and error detection.

#### The Physical Layer

Frames are sent from the datalink layer to physical layer. The physical layer converts these frames into physical signals for transmission over a media. It can be converted to electrical signal if the media is a wire, light signal for an optical cable and radio waves for air.

We've been able to go through the details of each layer and understand what they are made of. It's important to note that for a message receiver the journey starts from the physical layer through to the application layer where it is displayed.

##### sources:

- [OSI Model Explained](https://youtu.be/vv4y_uOneC0) by TechTerms on YouTube
- [Wikipedia](https://en.m.wikipedia.org/wiki/OSI_model)