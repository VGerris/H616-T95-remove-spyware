# Trojan_Horse_in_the_house

Deep cleaning a Chinese android box, to remove spy/mal-ware

# Model and hardware info

This tv box ([WeChip V10](http://www.wechipbox.com/wechip-v10-android-10-allwinner-h616-quad-core-6k-smart-tv-box-p-613.html)) happens to be a renamed/rebranded classic T95 android box. Based on photos online the PCB, components, case and OS look to be the same.

- H616 Allwinner SOC
- 4GB RAM (DDR3)
- 2.4GHz Wifi
- Ethernet
- ...(you can look that up, I am not a tech review channel)

## Open it up

The box is [full plastic](https://gitea.raspiweb.com:2053/Gerge/Trojan_Horse_in_the_house/src/branch/main/images), no screws. The base is held in with plastic clips and can be removed with some plastic shims (or metal srewdriver if you don't mind scratches). The board is held in with three screws, cheaping out by not fixing the fourth corner. The SOC has a heatsink that presses against a thermal pad and tiny metal heat spreader sheet in the top of the case. Based on testing outside the case with only the heatsink, the SOC gets burning hot to touch without even playing any video, just idle system with Termux.

## UART

There is an unpopilated UART connector next to the SD slot that I soldered a header to. The logs are [attached](https://gitea.raspiweb.com:2053/Gerge/Trojan_Horse_in_the_house/src/branch/main/logs), but there is not much useful info.

# Setup

This device was never used with any google account (thank goodness). There is no need to have anybody logged in on a shared device. Side-loading apk files is the way to go. Alternate app stores are also good if you trust the app creators ( [Fdroid](f-droid.org) is one good example).

# Holes in the Firewall !!!

The problems started pretty soon, considering the TV is barely used. The router started showing a lot of open Upnp ports created by the TV box. The ISP router can't disable Upnp, so it is clear as day that the router is also at fault. When blocked, the device created new holes every time it booted up. It was very desperate to phone home and let whoever was controlling it into the LAN.

# Where does it phone home to?

Using `tcpdump` on my OpenWRT test router I inspected where the TV box connects to.

Here is a list of connections made during the first 10 minutes from boot.
```
1e100.net           google safe browsing
router              (DHCP and other communications)
IPV6                Neighbor solicitation and advertisement
mcast.net           igmp
ARP                 IPv4 stuff
224.0.0.251         Multicast DNS
45.87.78.35         NTPv3

Malicious:
103.235.47.161                                          hong kong ??? lighttpd server
209.100.149.34.bc.googleusercontent.com                 (not secure)
62.219.107.34.bc.googleusercontent.com                  (not secure)
ec2-44-239-109-225.us-west-2.compute.amazonaws.com      redirect to location.services.mozilla.com (trying to get location from networks)
139-177-186-19.ip.linodeusercontent.com                 (chinese)saletracker login page

```

NOTE: the `tcpdump` log was more than 1000 lines long (keep in mind for later)

# H616 is a curse

There are not many usable resources for H616 based devices (orange pi zero 2 uses it, but those images do not work on TV boxes), the H6 version is a lot better and has many community systems and linux distros available to run. I gave it a look anyway, connected to UART and looked at possible ways in. U-boot seems to be well locked, the only entry point seems to be a firmware upgrade check for USB media, however in the absence of a usable system image it is no use. Even worse is that some other TV boxes can apparently run a secondary system from SD card while preserving the original chinese Android frimware on the internal MMC, this does not seem to be the case here (I [tried this](https://forum.armbian.com/topic/26813-here-is-some-hopes-for-h616-tv-box/))

# USB Debugging

Working from a computer over ADB is a lot better than running a terminal in android. USB debugging can be turned on the usual way, but there is another option below that to enable device mode on USB0. This turns out to be the USB port nect to the SD card slot labeled USB1 on the side. When the device mode is enabled the power on that port is cut (peripherals will not work), this lets you safely connect it to the computer without back feeding power into your USB. It will appear as a normal android device, called `V10` in my case, letting you to connect over ADB.

# How to fix

There is a [great article](https://www.malwarebytes.com/blog/news/2023/01/preinstalled-malware-infested-t95-tv-box-from-amazon) about the T95 box with step-by step guide on how to remove the malware. It involves a factory reset, but fairly straight forward. The only interesting trick is the removal of `CoreJava`. In this case we can assume that these devices are the same based on hardware and software similarities.

# I have root, and I am not affraid to use it

Nice bonus to have root privileges when running adb, but also frightening news that malware can do the same. (there is a "root switch" in settings). Time to get to work cleaning the `/system/app/`, `/system/preinstall/` and `/system/priv-app/` directories. Researching the package and app names of the apks making sure not to delete essential parts of android, a lot of bloat can be removed. In fact this is no different from an ordinary android debload, apart from the fact that root allows permanent deletion of the apps to make sure they are never restored even on factory reset. Most notable malware is `com.adups.fota`, but in this case `CoreJava` and `DGBLuancher` were bigger threats.

There is the file system way to remove apps permanently (enetring `su` then removing `rm -r /system/app/adups`) or with `pm` (package manager) command until next reset.

`pm uninstall com.package.name` (used this for packages that I am sure will never need)

`pm disable-user --user 0 com.package.name` (Used this for allwinner packages that may be hardware related)

From the preinstalled apps I only kept Kodi and chrome, I have no need for any app stores or streaming services.

# Replacing parts that were removed or disabled

It is time to transfer apks over USB
- New launcher (try out a few)
- Activity Manager (to create settings shortcut, as there is no app launch icon)
- VLC and Jellyfin, maybe also update Kodi

Of-course not everything can be open source:
- New browser
- Android Sytem Webview (needs to be updated, because nothing will work)

# Clean and Secure

After removing the malware I ran another `tcpdump` test to see if it has really solved the problem. Right away, it is clear that the box is mostly quiet after boot. There were [24 lines of logs in total](https://gitea.raspiweb.com:2053/Gerge/Trojan_Horse_in_the_house/src/branch/main/logs/TCPdump_clean.log). This consisted of DHCP, NTP, and google safe browsing. So it is safe to say that the solution works. Keep in mind that after another reset the malware will return and the same process applies.

# Lessons learned

Never buy a cheap android box for its intended purpose, unless it is for hacking. In the case of H6 based models they are very useful little machines to install a custom distro on, but should never be used with the original firmware. There may be ways to clean the system like in this case, but honestly I would not be running the factory Android if I could install another OS.

Using on an isolated VLAN to stream local media is still possible with proper firewall zones. In this case a lot of functionality is lost (browsing internet, or any app that needs to go online).

The biggest problem is that most average users don't even notice anything wrong (not many people check the firewall or network traffic), even if they do "ignorance is bliss" because they can't fix it. Also, don't even think about asking them to isolate the device from the internet, because you will get shouted at for trying to take away the "convenient" malware infested lifestyle they got used to.

Next time you see a household with one of these Trojan Horses in the living room, make sure to let the owner know and offer assistance.
