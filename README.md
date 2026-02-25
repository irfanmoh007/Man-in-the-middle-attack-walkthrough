# Man-in-the-middle-attack-walkthrough
# Intercepting Network Traffic: A Deep Dive into ARP Spoofing and Man-in-the-Middle Attacks

When you connect to a public Wi-Fi network at a coffee shop or a hostel, you share that digital space with dozens of strangers. We are often told not to log into sensitive websites on these networks, but what does the actual threat look like? 

To understand the mechanics of local network interception, I set up a controlled Man-in-the-Middle (MitM) lab using my Kali Linux virtual machine. My goal: bypass the network's normal routing and capture the raw login credentials of a smartphone connected to the same Wi-Fi.

Here is exactly how I executed the attack using ARP Spoofing, and why unencrypted websites are so dangerous.

## The Concept: What is ARP Spoofing?

[Image of ARP spoofing man-in-the-middle attack diagram]

To capture a device's traffic without specialized hardware, you have to force that device to send its data *through* your computer before it reaches the Wi-Fi router. We do this by exploiting a protocol called **ARP (Address Resolution Protocol)**.

ARP is how devices figure out the physical MAC addresses of other devices on the local network. The fatal flaw in ARP is that it is completely stateless and overly trusting. 

**The Analogy:** Imagine you live in an apartment building. If a random person walks up to your door and says, *"Hey, I'm the new mailman, hand me your outgoing letters,"* you shouldn't blindly trust them. But in the world of ARP, devices do exactly that. They don't check IDs.

By sending out fake (spoofed) ARP replies, my Kali Linux machine told two distinct lies:
1.  **To the target smartphone:** *"Hey, my Kali MAC address is the real location of the Wi-Fi Router."*
2.  **To the Wi-Fi Router:** *"Hey, my Kali MAC address is the real location of the smartphone."*

Both devices instantly updated their internal directories. The network was poisoned, and I was officially the Man-in-the-Middle.

## The Execution: Command Line Breakdown

Here are the exact commands used to pull this off, and the anatomy of what they actually do to the Linux operating system.

### Step 1: Turning the Attacker Machine into a Router

If I intercepted the traffic without knowing what to do with it, the target phone would simply lose internet access. To silently spy on the data, I had to tell my Kali Linux kernel to act like a router and forward the packets along.

<img width="745" height="146" alt="Screenshot 2026-02-25 123255" src="https://github.com/user-attachments/assets/0a3af6d4-58f1-4ac2-be56-0fe3092f0d96" />

sudo sysctl -w net.ipv4.ip_forward=1

sysctl: The system control tool used to modify the core brain of the Linux operating system (the kernel) while it is running.

-w: Tells the system to "write" or overwrite a kernel setting.

net.ipv4.ip_forward=1: By changing this value from 0 (off) to 1 (on), it flips an internal switch. Now, when a packet arrives that isn't meant for my Kali machine, instead of dropping it, the kernel politely forwards it to its true destination.

Step 2: Poisoning the Network
With routing enabled, I opened two terminals and used the arpspoof tool to broadcast my lies. My Kali network interface was eth0, the router was 192.168.100.1, and the target phone was 192.168.100.140.

Terminal 1 (Lying to the Phone):

Bash
sudo arpspoof -i eth0 -t 192.168.100.140 192.168.100.1
-i eth0: Specifies the network interface to shout the lies from.

-t 192.168.100.140: The Target. I am aiming this packet directly at the smartphone.

192.168.100.1: The Stolen Identity. I am pretending to be the router.

Terminal 2 (Lying to the Router):

Bash
sudo arpspoof -i eth0 -t 192.168.100.1 192.168.100.140
Translation: Aimed at the Router (-t 1), claiming to be the Phone (140).

Proof of Poisoning: Wireshark immediately throws duplicate IP warnings, confirming the network's ARP tables have been successfully poisoned.
<img width="1200" height="128" alt="Screenshot 2026-02-25 123528" src="https://github.com/user-attachments/assets/fa33dfbe-92ac-46ef-8db3-6ec46e89816b" />

The Capture: Seeing the Invisible
With the trap active, all traffic from the smartphone was flowing through my Kali machine.

To demonstrate the impact, I opened a deliberately vulnerable, unencrypted testing website (http://testphp.vulnweb.com) on the target smartphone and submitted a fake username (Ccccg) and password (ggg).

Over on my Kali machine, I had Wireshark running, passively listening to the flow of traffic. I applied a strict filter to cut through the background noise and hunt for the exact login submission:

Plaintext
http.request.method == "POST"
(Note: In web traffic, a GET request just loads a page, while a POST request submits form data to the server).

<img width="1919" height="1079" alt="Screenshot 2026-02-25 122604" src="https://github.com/user-attachments/assets/90e560e5-5439-4afe-ac41-de6bbea5e458" />

The Result:

Because the website used standard HTTP instead of secure HTTPS, there was absolutely zero encryption protecting the payload. My Kali machine captured the raw POST request right out of the air. As you can see in the packet capture, the submitted credentials (uname=Ccccg and pass=ggg) are visible in absolute plain text.

The Takeaway
Running this lab solidifies exactly why network security protocols exist.

Trust is a Vulnerability: Local networks inherently trust the devices connected to them. ARP spoofing is trivial to execute and instantly compromises that trust.

Encryption is Non-Negotiable: If that login portal had used HTTPS (TLS encryption), I still would have intercepted the packet, but the contents would have been scrambled, mathematical gibberish. Because it was HTTP, stealing the password was as easy as reading a postcard in transit.

Never trust a public network, and always look for the padlock in your browser.


Disclaimer: This documentation is for educational purposes only. All testing was performed on a localized, self-owned lab network with explicit authorization. Unauthorized interception of network traffic is illegal.
