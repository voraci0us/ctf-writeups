# Pandora - Network Traffic Analysis

Fun challenge, custom protocols are always interesting. Main challenge here is finding the traffic.

_ | _
:-------------------------:|:-------------------------:
![image](https://user-images.githubusercontent.com/37031448/188222306-85456ff0-d54f-4bdb-b0b1-e6bcb74b4ac5.png)|![image](https://user-images.githubusercontent.com/37031448/188222341-0332c260-689c-4548-96f5-b5f3140a9108.png)

First we need to locate the custom protocol traffic. The documentation does not mention what OSI layer the protocol runs on or what protocols support it.
Let's take a look at the protocol hierarchy for clues (Statistics -> Protocol Hierarchy):
![image](https://user-images.githubusercontent.com/37031448/188222899-3d0f48b4-a2cc-4332-85a9-ca414329f60e.png)

Only a few of these have enough packets to be what we're looking for:
- SSH
- HTTP
- DNS
- TCP Data
- ICMP

Note: Wireshark labels TCP traffic as "Data" when there is a payload portion exists but is not recognized as a known protocol.


I decided to go through TCP streams (right-click a TCP packet -> Follow -> TCP Stream, use up/down arrows by number in bottom right to browse) for anything interesting. SSH traffic looked like SSH traffic, HTTP traffic was boring (mostly browsing cyberskyline.com, the platform NCL is hosted on).
Stream 56 looked interesting, so I stopped to investigate:
![image](https://user-images.githubusercontent.com/37031448/188223624-6872243e-b110-409f-b998-cbe8182bbe6f.png)

Filtered the packets to show only this TCP stream, and only packets that contain a payload (data.data):
![image](https://user-images.githubusercontent.com/37031448/188224076-e87f62d5-99b4-441a-90cc-59d7fd0a7662.png)

These lengths match up with the protocol description - in retrospect `data.len == 4` would have found this quicker.
We can solve most of the CTF questions from here. Client initates the conversation, so the source IP of the first packet is the client and the destination IP is the server. Destination port aka "port the server is listening on" can be obtained from the first packet as well. 

Here's the protocol again for reference:

![image](https://user-images.githubusercontent.com/37031448/188224532-80383280-0770-4c5c-8ef1-9ff4d211debb.png)

According to the protocol reference, the "magic 2-byte ID" should be the second packet the client sends: 0x0417 = 1047.
The first packet the client sends specifies the number of encrypt requests: 0x5 = 5.
The length of first request is supposed to be a 4-byte (8 hex digts) number: 0x00000058 = 88.

Now we need the size of an individual encrypted hash. The first server response is supposed to contain the total encrypted length: 0xa0 = 160. Since we know five encryption requests are made, each response should be 160 / 5 = 32 bytes. This matches up with the server response sizes (32 + 128 = 160)!

We now know that the first encrypt response is the first 32 bytes the server sends back (ignoring the first packet the server sends, the encrypt length). This is the entire contents of packet #4 in the screenshot. The second encrypt response will then be the first 32 bytes of packet #5.

Finally, we need to find the hidden flag being transmitted. Since we have both the plaintext and the ciphertext, I am assuming it is more useful to look at the ciphertext.
![image](https://user-images.githubusercontent.com/37031448/188225850-c4b42bab-caab-4373-8413-3c9d8d698262.png)

I follow the TCP stream, show only one side of the conversation, and display as raw. The higlighted text is the packet containing the plaintext messages. There's a repeating pattern of `000000` periodically - this corresponds to the 4-byte length of the requests. So if we get rid of `00000058`, `00000048`, `0000006b`, and so on, we are left with a contiguous block of plaintext:

```546b4e4d4c555a4b513063744d54597a4d69424f51307774526b7044527930784e6a4d794945354454433147536b4e484c5445324d7a4967546b4e4d4c555a4b513063744d54597a4d69424f0a51307774526b70445279300417784e6a4d794945354454433147536b4e484c5445324d7a4967546b4e4d4c555a4b513063744d54597a4d69424f51307774526b7044527930784e6a4d79494535440a54433147536b04174e484c5445324d7a4967546b4e4d4c555a4b513063744d54597a4d69424f51307774526b7044527930784e6a4d794945354454433147536b4e484c5445324d7a4967546b4e4d0a4c555a4b513063744d54597a4d69424f51307774526b7044527930784e6a4d7949453544041754433147536b4e484c5445324d7a4967546b4e4d4c555a4b513063744d54597a4d69424f513077740a526b7044527930784e6a4d794945354454433147536b4e484c5445324d7a4967546b4e4d4c555a4b513063744d540417597a4d69424f51307774526b7044527930784e6a4d7949453544544331470a536b4e```

Converting from hex to ASCII then base64 decoding reveals the flag:
![image](https://user-images.githubusercontent.com/37031448/188226416-aad5feeb-12fa-46f0-a73c-c3158c17b30c.png)

I used https://gchq.github.io/CyberChef/ but you can also do this in bash or whatever tool you prefer:
![image](https://user-images.githubusercontent.com/37031448/188226572-2cacc336-f00a-4182-840d-51c5b23c266f.png)

That's a wrap, folks.



