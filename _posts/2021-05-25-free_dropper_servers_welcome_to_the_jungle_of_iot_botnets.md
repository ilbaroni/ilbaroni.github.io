---
layout: post
title:  "Free Dropper Servers - Welcome to the jungle of IoT botnets"
date:   2021-05-25
categories: reversing
---

## Introduction

For this one, I decided to write about something a bit different than malware reversing. Of course, it's somehow connected but that's not the point! :)

So, we all know that IoT botnets are a wild west and that there are lots of devices with little security, and yet they still somehow get exposed on the internet. 🤷

I was tracking an IoT botnet, and recently there was a new update (new builds, new c2, new dropper servers, etc) and I came across a new sample that had a new dropper server configured in the code:

![ ](/assets/images/free_dropper_servers_welcome_to_the_jungle_of_iot_botnets/image-20210525154008046.png)

Normally, the dropper servers are regular cheap droplets that are used to distribute the main bot but this one was different. Instead, it was an exposed Cambium Networks PMP 450x administration panel (I guess these are like WIFI access points):

![ ](/assets/images/free_dropper_servers_welcome_to_the_jungle_of_iot_botnets/image-20210525154838556.png)

At this point, I had to guess how these administration panels could be used to distribute malware samples and after poking around I found an interesting endpoint, which was: `/upload.html`

![ ](/assets/images/free_dropper_servers_welcome_to_the_jungle_of_iot_botnets/image-20210525155229469.png)

Ok... file upload. Very nice, but that's not all.

After searching on Shodan, I found that this misconfigured appliance besides the webserver was also exposing FTP and telnet.

Inside the telnet interface:

![ ](/assets/images/free_dropper_servers_welcome_to_the_jungle_of_iot_botnets/inside_telnet.png)

Full list of commands provided by the telnet interface:

```
Telnet+> help
Command Line Functions
addwebfile -- Add a custom web file
anticloning_status -- Display anticloning key status
arp -- ARP table entries
bcb -- Print BridgeCb
bermodap -- BER modulation configuration for Catalina embracing AP's
bertoff -- BERT test on/off, berton/bertoff
berton -- BERT test on/off, berton/bertoff
btbl -- Print Bridge Table
burnflash -- Burn flash from file
capt -- Read long word from memory
captmask -- Read long word from memory
cfgload -- Import an instrument configuration from file
cfgsave -- Save the instrument configuration to a file
clearsyslog -- Clear the system event log:
clearwebfile -- Clear all custom web files
clrbtbl -- Clear BridgeTbl
clrrmtsyslog -- Clear remote device system log
curl -- cURL command to fetch/upload data
debounceonboard -- Enable/Disable Debounce on On-board GPS
debouncetimingport -- Enable/Disable Debounce on Timing Port
debouncepowerport -- Enable/Disable Debounce on Power Port
defaulttxpower -- Set the default Tx output power for the radio
dfscaldump -- DFS Calibration RAM & Flash dump
dhcplog -- DHCP info log
dis900freqflg -- Disable Frequency Band Limit on 900 MHz Radios
en900freqflg -- Enable Frequency Band Limit on 900 MHz Radios
engreset -- Reset the unitexit -- Exit from telnet session
fpga_conf -- Update FPGA program
fpgainfo -- FPGA image info
fpgaversion -- FPGA version string
fskcaldump -- FSK Calibration RAM & Flash dump
fsksetdefaults -- FSK Calibration set to defaults
ftp -- File transfer application
gpsinfo -- GPS Status
gethex -- Retrieve a 4 bytes hex value
getledmode -- Query the Radio (SM or BHS)to get the current mode of LED Operation
help -- Command line function 
helpicmpstat-- ICMP layer stats
idlecnt -- Tick count since last Idle Task switch
igpstest -- Tests the IGPS unit for P12 modules
ip -- IP address modify/display
ipconfig -- IP configuration
ipstat -- IP layer stats
jbiflash -- Update FPGA program
jitterfilter -- Enable/Disable 1PPS Jitter Filter
kfactor -- Set the K-factor for the radio
lbtmenu -- Listen Before Talk for 3.6 GHz
LinkQual -- LinkQual: performs link quality test
ls -- List all files usage ls , ls -l
lsweb -- List Flash Web files
mac -- MAC address modify/display
maxtxpower -- Set the maxtxpower for the radio
minmaxtxpower -- Set the minmaxtxpower for the radio
minsw -- Minimum Software Version Required
mintxpower -- Set the mintxpower for the radio
muxswap -- Adjust the internal muxes for the radio
netgateway -- Default Network Gateway IP Address modify/display
netmask -- Network Subnet Mask modify/display
newdfscal -- DFS calibration for OFDM & FSK
nomtxpower -- Set the nominal Tx output power for the radio
ofdmcaldump -- Catalina OFDM Calibration RAM & Flash dump
password -- Change user account password
ping -- Send ICMP ECHO_REQUEST packets to network hosts
pingend -- End ICMP ECHO_REQUEST packets to network hosts
pldversion -- PLD version string
pllfailcnt -- PLL fail cnt status
psh -- Execute batch file
reset -- Reboot the unit
resetdefault -- Reboot to default mode
rfcal -- RF Calibration Menu
rfcb -- Print RF Bridge Cb
rfoff -- Turn off the RF and reset the FPGA
rfofft -- Set the timeout for the RFOff command
rftelnetaccess -- Enable/Disable the fowarding of Telnet packets coming from the RF interface (AP ONLY)
rfsync -- force synchronization
rm -- Remove file (no wildcards). usage rm <filename>
route -- Display, Add, or Delete IP routes
rxcomp -- RX Temp compensation test menu
rxpower -- Set the rxpower A/D for the radio
rxpowerlevel -- Set the rxpower level for the radio
rxtest -- Receiver test setup
sesstatus -- Display the current session status
setclock -- Set the system date and time
setfreq -- Set Scan Frequencies
sethex -- Save a 4 bytes hex value
setpwrctrl -- Enable/Disable periodic power control while in rfoff
settestber -- Set test Rx BER - no reboot required
setrxmaxgain -- Set RX gain to MAX Gain - no reboot required
setrxmingain -- Set RX gain to MIN Gain - no reboot required
settestcwpwr -- Set Carrier Wave test xmit power - no reboot required
setlegacyledmode -- Config the Radio (SM or BHS)to use Legacy Mode of LED Operation
setrevisedledmode -- Config the Radio(SM or BHS) to use Revised Mode of LED Operation
settestfreq -- Set test frequency - no reboot required
settestmod -- Set test modulation - no reboot required
settestpwr -- Set test xmit power - no reboot required
settestrxber -- Configure Radio for RX BER measurements
settestrxtx -- Configure Radio for RX and TX measurements
settestsisocntrl -- Enable/Disable of siso control messages
settestxmit -- Set test xmit - no reboot required
silentcarrier -- Silent carrier ON/OFF
slope -- Set slope parameter for the radio
snmpv3stats -- SNMPv3 Statistics
snmpv3tables -- SNMPv3 Table Settings
softversion -- New software version string
synconboard -- Enable/Disable On-board GSP Sync
synctimingport -- Enable/Disable GSP Sync over Timing Port
syncpowerport -- Enable/Disable GSP Sync over Power Port
syslog -- display system event log:  syslog <optional filename>
telnet -- Telnet client application
tempcomp -- Enable/disable TX temperature compensation when setting TX power
tfsoff -- Disable tFTP Server
tfson -- Enable tFTP Server
txtest -- Transmitter test setup
uartbaudrate -- UART baud rate control
udp -- UDP layer stats
update -- Enable automatic SM code updating
updateoff -- Disable automatic SM code updating
uptime -- Up time of radio
useradd -- Add a new user account or modify admin access level
userdel -- Delete a user account
users -- List all users
ver -- Software version string
version -- Software version string
```

There are lots of cool options to abuse in this list... I will highlight here a few good ones:

- curl - cURL command to fetch/upload data
- ls
- rm - Remove file (no wildcards)
- telnet - Telnet client application
- psh - Execute batch file
- useradd - Add a new user account or modify admin access level
- userdel - Delete a user account
- users - List all users

## Misconfigured devices in the wild 

So far I was able to identify 123 misconfigured devices on the internet exposing the telnet interface.

![ ](/assets/images/free_dropper_servers_welcome_to_the_jungle_of_iot_botnets/image-20210525165418226.png)

Numbers of devices by country:

| Country       | Total |
| ------------- | ----- |
| United States | 101   |
| Canada        | 13    |
| Nigeria       | 4     |
| New Zealand   | 3     |
| Italy         | 2     |

Additionally, I was able to quickly test 114 IP addresses out of the 123 that were found exposing the telnet interface, and I confirmed that 107 have the arbitrary file upload endpoint exposed to the internet:

![ ](/assets/images/free_dropper_servers_welcome_to_the_jungle_of_iot_botnets/image-20210525172034179.png)

# Conclusion

Welcome to the jungle :)
