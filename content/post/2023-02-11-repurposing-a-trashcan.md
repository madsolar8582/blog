---
title: "Repurposing a Trashcan"
subtitle: ""
date: 2023-02-11T13:40:00-05:00
tags: ["apple", "mac"]
---

While the 2013 Mac Pro might not be the beast that it was back in the day, it can still accomplish quite a bit. Unfortunately, it no longer supports the latest version of macOS (Ventura at the time of writing) and Monterey is quite buggy. To rejuvenate this machine, you can install Windows, Linux, or ChromeOS Flex.

Out of the gate, Windows isn't going to last long as Windows 10 will reach End of Life in October of 2025. ChromeOS Flex is interesting, but limits you to a small pool of applications and features as this OS is geared towards lower powered devices. Linux offers a lot of options, so it can be daunting to choose the "right" distro. Luckily, I tried a few and ended up in a good spot. Note: you should at least install up to macOS 12.4 to ensure that the Mac Pro has the latest firmware (430.120.6.0.0) prior to installing an alternative OS.

#### AlmaLinux

AlmaLinux is based off of RedHat Enterprise Linux that is free to use (i.e. does not require a subscription). It's main selling point is rock solid stability and binary compatibility with all RedHat RPMs. Unfortunately, trying AlmaLinux 9.1 did not work for me as the installer failed to boot as it kernel panic'd repeatedly.

#### CentOS Stream

CentOS Stream is a rolling release that serves as a proving ground for features to land in RedHat Enterprise Linux. This allows you to get about the same level of stability while also getting new features sooner. Unlike the AlmaLinux installer which at least booted, the CentOS Stream 9.1 installed never got past the UEFI screen.

#### Fedora Workstation

Continuing with the trend of RedHat-based distros, Fedora Workstation is the bleeding edge. Features that bake in this release move onto CentOS after a while, so you get very quick iterations of new features and functionality, at the cost of "enterprise-grade" stability. Luckily, this installer booted and made it all the way to partitioning when it failed to write the bootloader to disk multiple times.

#### Manjaro

Having exhausted the RedHat-based offerings, I tried out Manjaro (Arch-based). The OS installed just fine and was surprisingly snappy. However, once in the desktop environment, I had applications not launch reliably, which was a problem as I couldn't use the system.

#### Pop!_OS

Moving on from Arch-based systems, I tried out Pop!_OS which is Debian-based and maintained by System76 (a Linux hardware manufacturer). Overall, the install experience was fine and it worked just fine. However, I did not enjoy how the desktop experience worked and I ran into some networking issues.

#### Linux Mint

Last on the list (and the OS I stuck with) is Linux Mint. Mint is Debian-based and has started to veer off from being in line with its parent OS Ubuntu. This is fine with me as Canonical (the company responsible for Ubuntu) has been pushing [Snaps](https://en.wikipedia.org/wiki/Snap_(software)) and those have received mixed results (Mint does not use Snaps by default). Of course, with the advent of [Ubuntu Pro](https://ubuntu.com/pro), Mint does not get livepatching. 

Regardless, Mint works great out of the box except for the one issue I ran into: Cinnamon CPU load. Cinnamon is the default desktop environment and it looks very nice. However, when leveraging remote sessions (VNC/xrdp), it spikes the CPU load ridiculously high across all cores. To solve this, I replaced Cinnamon with xfce4. This has the side effect of not looking as nice, but it uses far less RAM and is much nicer to the CPU.

Speaking of remote sessions, I ended up using xrdp as that allows for resizable sessions. The setup is pretty straightforward:

```bash
sudo apt update 
sudo apt install xrdp
sudo usermod -a -G ssl-cert xrdp
sudo systemctl restart xrdp
sudo ufw allow 3389 # This assumes you have a proper firewall on your LAN
sudo ufw reload
```

Once that is done, you can connect using the Remote Desktop app (macOS/iOS) or the built-in RDP client on Windows. What's nice about this is that I can connect using my iPad Pro and do work from anywhere in my house. Wouldn't it be nice if Apple released a ARD app for iPad...

---

Overall, these trashcans serve as decent servers and can be workstations for moderately complex workloads. Yes, a base M1 runs circles around the E5-1650v2 and uses way less power and yes, any modern GPU will crush the dual FirePro D500s but these machines still have life in them.

If you are adventurous, you can replace the CPU with a E5-2687Wv2 but beware of going above 64GB of RAM as anything above that slows down the bus speed.

```
             ...-:::::-...                 madison@Trashcan 
          .-MMMMMMMMMMMMMMM-.              -------------- 
      .-MMMM`..-:::::::-..`MMMM-.          OS: Linux Mint 21.1 x86_64 
    .:MMMM.:MMMMMMMMMMMMMMM:.MMMM:.        Host: MacPro6,1 1.0 
   -MMM-M---MMMMMMMMMMMMMMMMMMM.MMM-       Kernel: 5.15.0-60-generic 
 `:MMM:MM`  :MMMM:....::-...-MMMM:MMM:`    Uptime: 2 days, 5 hours, 42 mins 
 :MMM:MMM`  :MM:`  ``    ``  `:MMM:MMM:    Packages: 2179 (dpkg) 
.MMM.MMMM`  :MM.  -MM.  .MM-  `MMMM.MMM.   Shell: bash 5.1.16 
:MMM:MMMM`  :MM.  -MM-  .MM:  `MMMM-MMM:   Resolution: 2560x1440 
:MMM:MMMM`  :MM.  -MM-  .MM:  `MMMM:MMM:   DE: Xfce 4.16 
:MMM:MMMM`  :MM.  -MM-  .MM:  `MMMM-MMM:   WM: Xfwm4 
.MMM.MMMM`  :MM:--:MM:--:MM:  `MMMM.MMM.   WM Theme: Mint-Y-Aqua 
 :MMM:MMM-  `-MMMMMMMMMMMM-`  -MMM-MMM:    Theme: Mint-Y-Dark-Aqua [GTK2/3] 
  :MMM:MMM:`                `:MMM:MMM:     Icons: Mint-Y-Dark-Aqua [GTK2/3] 
   .MMM.MMMM:--------------:MMMM.MMM.      Terminal: xfce4-terminal 
     '-MMMM.-MMMMMMMMMMMMMMM-.MMMM-'       Terminal Font: Monospace 10 
       '.-MMMM``--:::::--``MMMM-.'         CPU: Intel Xeon E5-1650 v2 (12) @ 3.900GHz 
            '-MMMMMMMMMMMMM-'              GPU: AMD ATI Radeon HD 7870 XT 
               ``-:::::-``                 GPU: AMD ATI Radeon HD 7870 XT 
                                           Memory: 2038MiB / 32072MiB 

```
