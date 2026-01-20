---
date: "2025-11-15T18:21:02+01:00"
lastmod: "2026-01-20T16:13:10+01:00"
draft: false
categories: ["Android", "Networking", "Security"]
tags: ["network-capture", "packet-analysis", "tcpdump", "wireshark", "openwrt", "mitmproxy", "pcapdroid", "https-decryption", "port-mirroring", "vpn", "traffic-monitoring", "IoT"]
title: "Capture Network Traffic on Mobile Devices"
summary: "A guide to capturing and analyzing mobile network traffic using invasive methods (PCAPdroid, Mitmproxy) and non-invasive techniques (OpenWRT router with tcpdump, port mirroring, LAN tap). Includes HTTPS decryption and real-time monitoring with Wireshark."
---

Capturing network traffic on mobile devices presents initial challenges, but appropriate tools and methods make the process straightforward and applicable to other devices such as IoT devices. This post focuses on Android devices.

Capture can be performed using an invasive approach, where the system or applications are aware of the capture process, or a non-invasive approach, where capture occurs externally without modifying the device or applications.

## Invasive Method

With the Invasive Method, the applications being monitored can be aware of the capture process. When monitoring power consumption or hardware usage, this method introduces overhead due to the additional application running on the device.

### Packet Capture Application

One popular open-source application is [PCAPdroid](https://play.google.com/store/apps/details?id=com.emanuelef.remote_capture), which enables capturing network traffic without root access. It creates a local VPN on the device to capture all network traffic.

This method has several limitations:

- Some applications may not work properly when a VPN is active.
- The length of each captured frame packet can be inaccurate for some analysis; sometimes it may not show the single fragmented TCP packets but the reassembled packets as one.
- Not all traffic is guaranteed to be captured; some system applications may use techniques to bypass the VPN.
- ARP and other link-layer protocols are not captured; only IP traffic is captured.

For most use cases, this method provides adequate coverage and straightforward setup. PCAPdroid also supports capturing traffic from specific applications, which reduces the amount of data captured.

### Proxy

Using a proxy constitutes an invasive method, as the applications being monitored and the system are aware of the proxy settings due to the required device configuration.

A proxy can be configured at the router level instead of the device level, but this configuration may still be detected[^1].

A commonly used proxy is [Mitmproxy](https://mitmproxy.org/), which enables capturing and modifying network traffic. It supports HTTP and HTTPS traffic, as well as other protocols.

## Non-Invasive Method

The non-invasive method ensures that the applications and the system remain unaware of the capture process. This approach avoids introducing overhead on the device and can be applied to any network device, not just mobile devices.

The primary advantage of this method is its ability to capture all traffic, including ARP and other link-layer protocols. However, it cannot isolate traffic from a single application, as it captures all traffic on the network.

### OpenWRT Router with Tcpdump

A reliable method for capturing network traffic from mobile devices involves using a router with OpenWRT firmware and Tcpdump installed. This approach enables capturing all traffic on the network, including ARP and other link-layer protocols.

Tcpdump supports capturing traffic from specific devices by filtering based on MAC or IP addresses.

Multiple devices support OpenWRT firmware, and some ship with OpenWRT pre-installed, such as [GL.iNet routers](https://www.gl-inet.com/). The primary limitation of this method is the storage space on the router, as capturing all traffic can quickly consume available space. Capture can be performed remotely via SSH, and GL.iNet routers support USB and/or SD card storage expansion.

For extended capture sessions, implementing a system service that manages its lifecycle in case of failure is recommended.

Below is an example of a service file saved as `/etc/init.d/tcpdump_capture`:

```bash
#!/bin/sh /etc/rc.common

USE_PROCD=1
START=99

PROG=/root/myservices/tcpdump.sh   # Path to the traffic capture script

start_service() {
    procd_open_instance
    procd_set_param command $PROG
    procd_set_param respawn     # Automatically restart service if it crashes
    procd_set_param stdout 1    # Redirect standard output to system log
    procd_set_param stderr 1    # Redirect standard error to system log
    procd_close_instance
}
```

To enable the service, run the following commands on the router:

```sh
chmod +x /etc/init.d/tcpdump_capture
service tcpdump_capture enable  # Enable the service to start on boot
service tcpdump_capture start
```

The script `/root/myservices/tcpdump.sh` could be something like this:

```bash
#!/bin/sh

set -e  # Exit immediately if a command exits with a non-zero status

now=`date +"%Y-%m-%dT%H-%M-%S"`

# Saves to external SD card in pcap directory
out="/tmp/mountd/disk1_part1/pcap/capture-$now.pcap"
echo $out

/usr/sbin/tcpdump -i wlan0 -n -w $out host 192.168.0.10 or ether host xx:xx:xx:xx:xx:xx
```

This script captures all traffic from a specific device on the network, based on its IP or MAC address, and saves it to an external SD card in pcap format.

For real-time traffic viewing, the latest captured pcap file can be piped to Wireshark running on a remote machine:

```bash
eval $(ssh-agent -s)     # Start the SSH agent
ssh-add .ssh/gl_axt1800  # Add the private key for authentication

# SSH into the router, tail the latest pcap file, and pipe it to Wireshark
ssh -i .ssh/gl_axt1800 root@192.168.8.1 "tail -n +1 -f /tmp/mountd/disk1_part1/pcap/\$(ls -t /tmp/mountd/disk1_part1/pcap/ | head -n 1)" | wireshark -k -i -
```

These commands establish an SSH session to the router, tail the latest pcap file, and pipe it to Wireshark running on the local machine for real-time viewing. Using ssh-agent avoids entering the password multiple times. Without the agent, the last command requires password entry.

### Port Mirroring / SPAN

When a router with OpenWRT is not available, network traffic can be captured using a managed switch with port mirroring or SPAN (Switched Port Analyzer) functionality. This method captures all traffic from a specific port on the switch and forwards it to another port where a capture device is connected.

{{< center_image src="/images/posts/capture_network_mobile/port_mirroring.png" alt="Port Mirroring" caption="Port Mirroring" width="700px" >}}

Affordable managed switches supporting port mirroring are available, such as TP-Link switches. While the setup is straightforward, care must be taken to avoid creating network loops or duplicating traffic.

### Throwing Star LAN Tap

Network traffic can also be captured using a LAN Tap. These devices can be purchased or built. A LAN Tap is positioned between the monitored device and the network, enabling traffic capture without modifying the device or network. The [Throwing Star LAN Tap](https://greatscottgadgets.com/throwingstar/) from Great Scott Gadgets is one example of such a device.

{{< center_image src="/images/posts/capture_network_mobile/throwing_star_lan.png" alt="Throwing Star LAN Tap" >}}

## Decrypting HTTPS Traffic

Most captured network traffic is encrypted using HTTPS. To decrypt HTTPS traffic, a custom root certificate must be installed on the monitored device. This can be accomplished by exporting the certificate from the capture tool (e.g., Mitmproxy) and installing it on the device.

Recent Android versions require applications to be configured to trust user-installed certificates, as they only trust system-installed certificates by default. This configuration can be achieved by modifying the application's network security configuration using tools like [apk-mitm](https://github.com/niklashigi/apk-mitm).

On rooted devices, the custom root certificate can be installed as a system certificate, which will be trusted by all applications. Alternatively, tools like [Magisk Trust User Certs](https://github.com/NVISOsecurity/MagiskTrustUserCerts) simplify this process.

## Bibliography

[^1]: "Towards Aggregated Features: A Novel Proxy Detection Method Using NetFlow Data," 2020 IEEE 22nd International Conference on High Performance Computing and Communications, pp. 409-416, 2020. [IEEE Xplore](https://ieeexplore.ieee.org/document/9408011)