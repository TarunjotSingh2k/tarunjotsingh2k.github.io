From Private to Public: How NAT Bridges Your Network to the Internet
====================================================================

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HA8LD18e9n9H2-4d45oKPw.png)

[Reference](https://medium.com/@tarunjotsingh2k/from-private-to-public-how-nat-bridges-your-network-to-the-internet-60e7f74c8b2c?source=your_stories_outbox---writer_outbox_published-------------------------------------------)

by [TarunjotSingh2k](https://medium.com/@tarunjotsingh2k?source=post_page---byline--60e7f74c8b2c-----------------------------------------)




What is NAT? A Simple Guide to Network Address Translation
----------------------------------------------------------

Have you ever wondered how multiple devices in your home — your phone, laptop, smart TV — can all access the internet through a single internet connection? The technology that makes this possible is called **NAT**, or **Network Address Translation**.

Let’s explore what NAT is, the different types, and how you can configure it — including a quick example using **Hyper-V**.

What is NAT?
------------

As the name suggests, Network Address Translation is a process of _translating_ one IP address into another. More specifically, NAT is used to map one **public IP address** to **multiple private IP addresses**. This allows devices within a private network (like your home or office) to access the internet using just one shared public IP.

This translation process happens on _edge devices_ such as **routers** or **firewalls** — the gateways between your internal network and the internet.

**Types of NAT**

There are several types of NAT, each serving different purposes:

*   **Static NAT**
    Maps a single private IP to a single public IP. This is useful when a device (like a server) needs to be accessible from the internet consistently.
*   **Dynamic NAT**
    Maps a private IP to any available public IP from a predefined pool. It selects the public IP randomly and is often used in larger networks.
*   **PAT (Port Address Translation)**
    Also known as **NAT Overload**, this is the most common form of NAT. It allows many devices to share one public IP address by differentiating traffic using port numbers. This is the standard approach used in most home and small office routers.

**Why NAT is Important**

1.  **IP Address Conservation**
    IPv4 addresses are limited. NAT allows multiple devices to share a single public IP, reducing the demand for new IPs.
2.  **Improved Security**
    NAT hides your internal IP addresses from the outside world. External systems can only see the public IP, adding a basic layer of protection.
3.  **Simplified Internet Sharing**
    NAT makes it easy to share a single internet connection across multiple devices, whether at home, in an office, or even in a virtual lab.

Using NAT in Windows 10 and Windows Server 2016 with Hyper-V
------------------------------------------------------------

Microsoft introduced a **NAT-enabled virtual switch** in **Windows Server 2016** and **Windows 10** to simplify networking for virtual machines (VMs). This setup allows VMs to communicate with external networks using the **host machine’s IP address**, without requiring complex network configurations or additional adapters.

How NAT Works in Hyper-V
------------------------

Just like traditional NAT on a router, **Hyper-V NAT** modifies the IP headers of packets sent from your VMs, translating private IP addresses into the host’s public-facing IP and port. This enables:

*   **Internet access for VMs** using a single external IP.
*   **Enhanced security** by isolating VM internal IPs from the public network.
*   **Protocol and port translation**, allowing seamless communication across different networks and services.

> **Security Bonus:** Since NAT hides internal IPs from external networks, it acts as a basic firewall — preventing direct access to your VMs from the outside unless explicitly forwarded.

Creating a NAT Virtual Switch in Hyper-V
----------------------------------------

> **Note:** You cannot create a NAT switch via the Hyper-V Manager GUI. This must be done via PowerShell.

Here’s how to do it:

Step 1: Create an Internal Virtual Switch

```
New-VMSwitch -SwitchName "NATSwitch" -SwitchType Internal
```![captionless image](https://miro.medium.com/v2/resize:fit:1248/format:webp/1*64QNHA0jwiTpQINXEOIDcA.png)

Step 2: Assign an IP to the Virtual Switch

```
New-NetIPAddress -IPAddress 192.168.100.1 -PrefixLength 24 -InterfaceAlias "vEthernet (NATSwitch)"
```![captionless image](https://miro.medium.com/v2/resize:fit:1248/format:webp/1*rAGkze0qclxDe43ixmOebw.png)

Step 3: Create the NAT Network

```
New-NetNat -Name "NATNetwork" -InternalIPInterfaceAddressPrefix 192.168.100.0/24
```![captionless image](https://miro.medium.com/v2/resize:fit:1248/format:webp/1*eGi1ovpdfhwcW7Y0EPQy5g.png)

Step 4: Configure VM IP Settings

Assign your VMs an IP within the `192.168.100.0/24` range and set:

*   **Gateway**: `192.168.100.1`
*   **DNS**: e.g., `8.8.8.8` (Google DNS)

Your VMs can now reach the internet via the host’s connection — all thanks to NAT.

Summary
-------

Microsoft’s NAT virtual switch offers a clean, practical solution for VM internet access in lab and development environments. With minimal configuration and no extra hardware or bridges, it provides:

*   Simplicity in setup.
*   Better isolation of VM traffic.
*   Real-world testing conditions in a virtual lab.

If you’re running VMs on your workstation or server, NAT is a lightweight yet powerful way to enable connectivity.
