# NetworkWalks B082 - Week 1: Cybersecurity Lab Setup

This is my Week 1 submission for the NetworkWalks B082 internship. The task was to build an isolated virtual lab where I can practice penetration testing without touching any real network or device I don't own.

Everything here runs on my own laptop inside Oracle VirtualBox. The whole lab sits on a private virtual network, so nothing I do inside it can reach anything outside.

## What the lab looks like

Four virtual machines, one attacker and three targets:

| VM | Role | OS | RAM |
|---|---|---|---|
| kali-linux-2026.2 | Attacker | Kali Rolling (Debian 64-bit) | 2048 MB |
| kali-linux-2025.4 | Spare attacker box | Kali Rolling (Debian 64-bit) | 2048 MB |
| Windows 10 Victim | Target | Windows 10 64-bit | default |
| Android 9 Victim | Target | Android x86 9.0 r2 | default |

I kept the 2025.4 image around as a backup after I broke networking on my main box and wasn't sure I could fix it. Turned out I could, but having a clean fallback took a lot of the pressure off while I was experimenting.

![Lab overview](Full%20Cybersecuirty%20&%20Pentesting%20Lab%20Setup.png)

## Steps I followed

### 1. Installed VirtualBox and created the NAT Network

Before building any VM I set up the network they would all share. In VirtualBox this lives under Network, then the NAT Networks tab.

I created one called `NatNetwork` with an IPv4 prefix of `10.0.2.0/24` and DHCP enabled. I left IPv6 off because I don't need it and it just adds noise when I'm reading packet captures later.

The reason for NAT Network instead of plain NAT is important. Plain NAT gives each VM its own private tunnel to the internet, but the VMs cannot see each other. NAT Network puts them all on one shared subnet, so Kali can actually scan and reach the victim machines. That distinction cost me some time to understand and it is probably the single most useful thing I learned this week.

![NAT Network config](VirtualBox%20NATNetwork%20Config.png)

### 2. Built the Kali attacker VM

Imported the Kali VirtualBox appliance and then went through the settings.

System settings: 2048 MB base memory, boot order set to Hard Disk then Optical, PIIX3 chipset, I/O APIC enabled, hardware clock in UTC. I left UEFI and Secure Boot off since the Kali image expects legacy BIOS boot.

![Kali system config](Kali%20Linux%20System%20Config%201.png)

Under Acceleration I confirmed Nested Paging was on. Without it the VM crawls.

![Kali acceleration](Kali%20Linux%20System%20Config%202.png)

Network settings: Adapter 1 attached to NAT Network, name `NatNetwork`, adapter type Intel PRO/1000 MT Desktop (82540EM), and Promiscuous Mode set to Allow All.

Promiscuous mode matters here. By default the virtual NIC drops frames that aren't addressed to it, which means sniffing tools would only ever show me my own traffic. Setting it to Allow All lets Kali see traffic on the whole virtual segment, which is the entire point of having an attacker box.

![Kali network settings](Kali%20Linux%20Network%20Settings.png)

### 3. Built the Windows 10 victim

Same network setup as Kali. NAT Network, `NatNetwork`, Intel PRO/1000 MT Desktop, promiscuous mode Allow All. Keeping the adapter settings identical across every VM meant one less variable to worry about when something didn't work.

![Windows 10 victim network](Windows%2010%20Victim%20Network%20Settings.png)

### 4. Built the Android 9 victim

Created a new VM pointing at `android-x86_64-9.0-r2.iso`.

VirtualBox has no Android profile in its OS list, so I had to set the type manually to Linux, distribution Other Linux, version Other Linux (64-bit). If you leave it on the auto-detected guess the VM will not boot properly.

![Android 9 victim](Android%209%20Victim.png)

### 5. Verified the tooling

Confirmed Nmap was present and current on the Kali box before calling the setup done.

```
$ nmap --version
Nmap version 7.99 ( https://nmap.org )
Platform: x86_64-pc-linux-gnu
```

![Nmap check](Verify%20Nmap.png)

## Problems I ran into

### Kali could not reach the internet

This was the big one and it ate most of my time.

After booting Kali I ran a basic connectivity test and got nothing back:

```
$ ping 8.8.8.8
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
...
17 packets transmitted, 0 received, +15 errors, 100% packet loss
```

Every single packet failed. Opening Firefox to google.com did nothing either, it just threw a graphics warning and sat there.

I also tried guessing at a command to open the network settings, which taught me nothing except that `advanced network configurations` is not a real command:

```
$ advanced network configurations
advanced: command not found
```

![Ping failure](Fixing%20Kali%20Linux%20Connectivity%20Issue.png)

### What I did about it

I opened NetworkManager and edited the Wired connection 1 profile under IPv4 Settings. The method was already Automatic (DHCP), so I added a static address entry of `10.0.0.1` with a netmask of 24, gateway `10.0.0.2`, and a DNS server of `10.0.0.1`, then saved.

After that, pinging that address worked cleanly:

```
$ ping 10.0.0.1
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.038 ms
...
11 packets transmitted, 11 received, 0% packet loss
```

Zero packet loss. That confirmed the virtual NIC and the TCP/IP stack on the guest were both alive and responding, which ruled out the adapter itself being broken or disconnected.

![Kali network config](Kali%20Linux%20Network%20Config.png)

### What the interface actually shows

Checking the interface afterwards was the most useful thing I did all week:

```
$ ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 08:00:27:5a:87:bc
    inet 10.0.0.1/24 scope global noprefixroute eth0
    inet 10.0.2.3/24 scope global dynamic noprefixroute eth0
```

![IP address check](Check%20IP%20Address.png)

Two addresses on one interface. The `10.0.0.1` is the static entry I added by hand. The `10.0.2.3` is the real lease that DHCP handed out from the NAT Network, and it sits on `10.0.2.0/24`, which matches the subnet I configured in VirtualBox back in step 1.

Being honest about what this means: `10.0.0.1` is the machine's own address, so pinging it only proves the guest can talk to itself. It is a genuinely useful test because it isolates the fault, but it is not proof that internet access is fixed. The address that actually matters for reaching anything outside is `10.0.2.3`, and traffic out of that subnet should be leaving through the VirtualBox NAT gateway at `10.0.2.1`, not through the `10.0.0.2` gateway I typed in.

So my working theory is that the static entry I added is on a subnet that does not exist anywhere in this lab, and it is likely competing with the correct DHCP lease rather than helping it. The next thing I want to try is removing the static entry entirely, leaving the profile on pure DHCP, bouncing the connection, and then testing `ping 10.0.2.1` first and `ping 8.8.8.8` second. Testing the near hop before the far one would have saved me an hour if I had done it in that order from the start.

## Where things stand

Working:

- NAT Network created and all four VMs attached to it
- All VMs boot
- Kali has a valid DHCP lease on the lab subnet at `10.0.2.3`
- Promiscuous mode enabled across every adapter
- Nmap installed and verified at 7.99

Still open:

- Outbound internet from Kali is not confirmed yet, see the section above
- I have not yet run a scan from Kali against the Windows and Android targets, that is the first thing on my Week 2 list

## What I took away from this

Reading error output properly is worth more than changing settings and hoping. `Destination Host Unreachable` coming from `10.0.0.1` was the machine telling me it had no route, and the address in that message was the clue the whole time. I just didn't know how to read it yet.

Changing one thing at a time also matters. When I was stuck I started adjusting several settings at once, and then had no idea which change caused what.

## Repository contents

All screenshots were taken during the actual setup, nothing was recreated afterwards.

- `Full Cybersecuirty & Pentesting Lab Setup.png` - VirtualBox Manager showing all four VMs
- `VirtualBox NATNetwork Config.png` - NAT Network setup, 10.0.2.0/24 with DHCP
- `Kali Linux System Config 1.png` - Kali motherboard and boot settings
- `Kali Linux System Config 2.png` - Kali acceleration settings
- `Kali Linux Network Settings.png` - Kali adapter attached to NatNetwork
- `Windows 10 Victim Network Settings.png` - Windows 10 adapter settings
- `Android 9 Victim.png` - Android x86 VM creation
- `Fixing Kali Linux Connectivity Issue.png` - the failed ping to 8.8.8.8
- `Kali Linux Network Config.png` - NetworkManager IPv4 settings and the successful ping
- `Check IP Address.png` - `ip a` output showing both addresses on eth0
- `Verify Nmap.png` - Nmap version check

## A note on scope

This lab is entirely self-contained. Every machine in it is one I created and own, running on hardware I own, on a virtual network that does not touch anything else. Nothing here is pointed at a system I do not have permission to test.
