+++
title = "WSL Networking Sucks: Mirrored Mode vs Reality"

date = 2026-01-25
description = "A weekend-by-weekend breakdown of why mirrored networking in WSL2 keeps failing, what fixes actually work, and how to keep .local resolution alive without nuking your setup."
[extra]
featured = false
tags=["tech","wsl","linux","windows","networking"]
+++

Hello Friends,

In this blog I am going to rant about my many struggles with networking in WSL.

For those of you who don't know, WSL is Windows Subsystem for Linux. It lets you run a Linux environment directly on Windows without the overhead of a traditional VM.

We mostly talk about two kinds of hypervisors (well three, but for brevity let's just say two):

- Type 1 hypervisors run directly on the host hardware (Hyper-V, VMware ESXi).
- Type 2 hypervisors run on top of a host operating system (VirtualBox, VMware Workstation).

WSL runs via Hyper-V under the hood, yet it is not a traditional VM because it has really good interop with the host Windows operating system. I was a big fan of WSL. My main workstation is primarily for gaming and professional .NET work, so Windows is a virus that won't be getting out of my system anytime soon. WSL let me keep bleeding-edge Linux right alongside Windows. I spent hundreds of hours customizing it to the point where I was running full desktop environments inside WSL with GPU acceleration for AI workloads, running Android applications natively, and even doing winception by trying to run Wine.

But my love for WSL has been waning over the past years, and everyone keeps saying 202x is the era of desktop Linux because it absolutely is.

Microsoft has turned WSL into a buggy mess. GitHub issues are rarely addressed, releases are few and far between, and the annoying stale GitHub bot will close issues because the author doesn't keep responding to it.

What is even the point of open-sourcing it if only Microsoft employees are working on it?

Like Windows, it also feels like a vibe-coded slop fest.

## Weekend One: Mirrored networking meltdown

So last week I was trying to set up a web server inside WSL for testing a few things, but when I tried to access it from another device in my local network I could not. I knew the reason why since I had struggled with it before.

WSL2 uses a NAT-based network by default. It creates a virtual network interface on the host Windows OS and assigns a private IP address to the WSL instance. This setup allows WSL to access the internet through the host's network connection, but it also means that incoming connections from other devices on the local network are not automatically forwarded to the WSL instance. The host can access the WSL instance just fine, but nothing else can.

I could get around this by setting up port forwarding on the host, but I did not want to do that. I knew there was a mirrored networking mode that was introduced with Windows 11, which allows WSL to "mirror" the networking interfaces on the host. I had a faint memory of struggling with it before, but I decided to give it another go. I enabled the mirrored networking mode by adding it to the `.wslconfig` file in my user directory like so:

```wslconfig
[wsl2]
networkingMode=mirrored
```

After doing this, I restarted WSL and boom my internet stopped working. I tried finding solutions online and tried debugging everything from top to bottom. Essentially, my `/etc/resolv.conf` had some gibberish values. Mind you, this had never happened before and a normal user should not be messing with this file.

I could disable this override by adding this to my `/etc/wsl.conf` file:

```conf
[network]
generateResolvConf = false
```

Then I removed the symlink to the resolv file and recreated a fresh one with public DNS servers, and so I did. Guess what? Internet was still broken. I started triaging this issue even more which led me to a rabbit hole of Reddit threads and GitHub issues. Turns out that mirrored networking mode is completely broken for a lot of people, with no known fixes. Somebody was advising people to reset the Windows network stack like so:

```powershell
netsh winsock reset
netsh int ip reset
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

Others suggested restarting the LxssManager service:

```powershell
Restart-Service LxssManager
```

Or restarting the Hns service:

```powershell
Restart-Service hns
```

Or running DISM repairs or ultimately reinstalling WSL. I COULD NOT do that. I had years' worth of customizations on this VM, and I have been moving this around across PCs to preserve it. This is what I dislike about Windows. I understand that having such a colossal titan of a codebase will result in bugs, but every Microsoft community answer leads to "Reinstall Windows," just nuke the whole thing, because I am sure not a single person understands Windows internals at Microsoft.

In the past 1 year I have had so many issues with Windows that I could write a book about it:

- random BSODs
- lag
- some "Microsoft" process using 99% of CPU and Disk
- Bluetooth audio breaking
- `localhost` not working courtesy of an update
- just 3 days ago Windows apps like Terminal and Notepad were broken on majority of the systems worldwide because of a "security" patch

*Can't have security issues if your computer doesn't work - teehee wink ;)* I am honestly so done with this. Microsoft needs to get their act together.

Coming back to WSL, I had already spent about 3 hours of my weekend debugging this and my OCD would not let me rest.

After many hours of debugging I found out the culprit to be `systemd`. By default WSL distributions have systemd disabled, but I had it enabled via `wsl.conf`:

```conf
[boot]
systemd=true
```

I also do not remember why I did that. I think it was because I was trying to set up avahi on WSL for `.local` hostname resolution. Disabling systemd made the internet work again, even with the defaults. But I started experiencing lag in ping commands and general network instability; sometimes the internet would just stop working randomly. [I am not alone](https://github.com/microsoft/WSL/issues/11369). I eventually gave up and switched back to NAT mode.

## The next weekend

I was trying to ssh into my homelab using the `.local` command, but it kept failing. Hmm what could have gone wrong? It had to be something from the last weekend's fiasco. Direct IP was working fine, but it is annoying to rely on it and I want everything to work the way I want it to, period.

After checking everything I did (god bless the history command), it turns out I had disabled systemd and avahi depends on it. One would think it would work after I enabled systemd, correct? IT DID NOT. I was already frustrated as f*ck.

I again tried looking at what I did while doing a thousand restarts, turning services on and off, setting firewall rules and what not. But one thing I did which was out of reach of the `history` command (which is the reason I did not find it earlier) was removing an entry from my `.wslconfig`:

```wslconfig
[wsl2]
dnsTunneling=false
```

DNS tunneling is a feature in WSL2 that allows it to use Windows APIs directly for name resolution. It is only supported in WSL2 and Windows 11 22H2 and above (don't quote me on that; I don't remember Windows versions well). Turns out it is enabled by default and thanks to a gentleman on [Reddit](https://www.reddit.com/r/bashonubuntuonwindows/comments/1e9rjid/mdns_not_working_in_wsl2_works_ok_in_windows_host/) I found out that it is required for `.local` addresses to resolve. Now I have no idea why that is; networking was my weakest subject in college. But adding this back made it start working again. I was annoyed that I had to do this, so I set off on a journey to dig the following two gemstones:

1. Have `.local` resolution
2. Have mirrored networking mode enabled

I mean how hard can it be, right? After many hours of trial and error I have found several ways of doing these with options if you want one or the other.

### Method 1: Networking Mode NAT and DNS Tunneling disabled with Port Forwarding

This is the easiest method to just get `.local` resolution to work. But this comes with the downside of other devices not being able to access services running inside WSL without port forwarding on the host.

1. Ensure your `.wslconfig` has the following entries:

```wslconfig
[wsl2]
networkingMode=nat
dnsTunneling=false
```

That's it pretty much, restart WSL and you should be good to go.

2. Identify the WSL IP. First, get the internal IP address of your WSL instance:

```powershell
wsl -d <DistroName> hostname -I
```

*(Usually something like `172.x.x.x`)*

3. Set up the port forward (Windows to WSL). Use `netsh` to map a Windows port to the WSL IP.

*Assuming your Node.js app/container is on port `3000`:*

```powershell
netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=<WSL_IP>
```

4. Open the Windows Firewall. Allow incoming traffic to that port so external devices can reach it:

```powershell
New-NetFirewallRule -DisplayName "NodeJS WSL" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3000
```

### Method 2: Networking Mode Mirrored and editing `hosts` file

1. Set the networking mode to mirrored in `.wslconfig`:

```wslconfig
[wsl2]
networkingMode=mirrored
```

`dnsTunneling` is on by default. This step alone allows other devices to access services running inside WSL.

2. Edit the WSL hosts file. Add the `.local` entries manually to your WSL `/etc/hosts` file:

```bash
sudo vim /etc/hosts
```

Add entries like:

```
IP_ADDRESS HOSTNAME
```

This comes with the downside of having to manually update IP addresses if they change. Anyway, this is the best option if you can set static IPs on your router.

### Method 3: Networking Mode Mirrored with Avahi and DNS Tunneling enabled

1. Set the networking mode to mirrored in `.wslconfig`:

```wslconfig
[wsl2]
networkingMode=mirrored
```

2. Now comes the tricky part. Since most of the time the broadcast is done via announcement and not by your router you need an mDNS resolver. Install `avahi` and `libnss-mdns` in your WSL distro:

```bash
sudo apt update
sudo apt install avahi-daemon libnss-mdns
```

Check if the following config file is correctly set up:

```bash
sudo vim /etc/nsswitch.conf
```

It should look like this:

```
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files
group:          files
shadow:         files
gshadow:        files

hosts:          files mdns4_minimal [NOTFOUND=return] dns # THIS IS THE IMPORTANT LINE
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files
```

The issue now comes with systemd. In any modern Linux distro systemd is the de facto standard for running services, and since `systemd` breaks internet we will use `avahi-daemon` directly:

```bash
sudo avahi-daemon --daemonize
```

This will result in a dbus error, because avahi uses a dbus backend and WSL does not have dbus running by default. You can run dbus first then run avahi:

```bash
sudo mkdir -p /run/dbus
sudo dbus-daemon --system
```

But I recommend editing the `/etc/avahi/avahi-daemon.conf` file to disable the dbus support:

```bash
sudo vim /etc/avahi/avahi-daemon.conf
```

Find the line `use-dbus=yes` and change it to `use-dbus=no`, then restart avahi:

```bash
sudo avahi-daemon --daemonize
```

This is enough to get both of the gemstones I was looking for, but this caused a lot of lag on the network, like unbearable kind of lag.

I solved it by providing my own `resolv.conf` file:

```bash
sudo rm /etc/resolv.conf && sudo vim /etc/resolv.conf
```

Add the following lines:

```conf
nameserver <YOUR ROUTER IP>
nameserver 8.8.8.8
nameserver 1.1.1.1
```

You can find your router IP by running `ipconfig` on Windows and looking for the default gateway of your main network adapter. After this prevent WSL from overwriting your `resolv.conf` by adding this to your `/etc/wsl.conf` file:

```conf
[network]
generateResolvConf = false
```

Restart WSL and you should be good to go!

## Conclusion

I need 2 weekends back, I am done with this OS. I will be switching to a full Linux desktop as soon as I can be bothered to reinstall everything. Wine is already making great progress on Adobe support (not that I care, I use Resolve and Affinity products now). WSL has been an incredible tool but I can't see it being ripped apart like this anymore.

If you are a developer and want a Linux environment, just dual boot or go full Linux. It's the year of the Linux Desktop after all.

