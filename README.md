# LoRaWanGateway
A LoRaWan Gateway in Lua, listening to all spreading factors on one specific channel.

## Goals
+ Port the OneChannelGateway to NodeMCU/Lua (https://github.com/things4u/ESP-1ch-Gateway) - Done!
+ Add support for sending messages to nodes - Done!
+ Add support for OTAA - Done!
+ Add support for receiving multiple SF's on a channel - Done!
+ Add support for more than one channel (won't work :-( )
+ Add support for remote access - Done!

## Why Lua?
Lua is an event driven language, which is a good match for a gateway: a gateway has to respond to 
incoming messages from nodes and routers. The LoraWanGateway runs on an ESP8266 chip, with the NodeMCU 
firmware it runs a full WiFi stack and a Lua runtime. The NodeMCU contains a lot of libraries that are 
usefull for a gateway: WiFi, NTP support, SPI, 6 timers, JSON support etc.

## How multiple spreading factors are detected
Unlike complex LoRaWAN concentrator modules (such as SX1301/SX1308), a simple SX1276/SX1278 module needs 
to be told exactly for which channel and spreading factor (SF) it should perform channel activity detection 
(CAD). This works fine when using the module in an end device (node), as then LoRaWAN explicitly defines 
which combination is to be used. But for a single channel gateway the goal is to listen on all SFs of that 
single channel.

The strategy to make this work for a simple SX1276/SX1278 module is to first start a SF7 CAD on the channel 
frequency. While performing the CAD, RSSI is used to detect the presence of a signal on the channel.

If a preamble is detected the message is received using the RX_SINGLE mode of the module. If a preamble is 
not detected on SF7 and RSSI shows that a signal is present, CAD is used to detect a SF8 preamble. This is 
repeated for all spreading factors.

This works because the preamble always uses 8 symbols, and the length of a symbol is dependent on the spreading 
factor. The next SF's symbol length is twice the length of the previous length.

Detecting a signal using CAD takes less than two symbols. So when a SF7 CAD returns, two symbols of the preamble 
are 'used', leaving 6 more to synchronize the reading of the message. The two SF7 symbols that are gone, acount 
for _one_ SF8 symbol. So there are 7 preamble symbols available to detect a SF8 signal. Likewise:

- For a SF9 signal there will be 8 - (1 + 0.5) = 6.5 symbols available
- For a SF10 signal there will be 8 - (1 + 0.5 + 0.25) = 6.25 symbols available
- For a SF11 signal there will be 8 - (1 + 0.5 + 0.25 + 0.125) = 6.125 symbols available
- For a SF12 signal there will be 8 - (1 + 0.5 + 0.25 + 0.125 + 0.0625) = 6.0625 symbols available

So there are always enough preamble symbols left to try to detect a higher SF signal, if the lower SF detection 
fails.

In order to make this work it is important to detect the presence of a signal on the channel as soon as possible; 
the RSSI detection during CAD is used to do that. A drawback of this approach is that the range of this gateway 
will be less of that of a 'real' gateway; it can only receive messages that can be detected by RSSI.

## Hardware
In order to run a LoRaWanGateway you need:

+ An ESP8266 module
+ An SX1276/SX1278 module
+ A way to flash your ESP8266

Connections
<table>
<tr><th>ESP PIN</th><th>SX1276 PIN</th></tr>
<tr><td>D1 (GPIO5)</td><td>DIO0</td></tr>
<tr><td>D2 (GPIO4)</td><td>DIO1</td></tr>
<tr><td>D5 (GPIO14)</td><td>SCK</td></tr>
<tr><td>D6 (GPIO12)</td><td>MISO</td></tr>
<tr><td>D7 (GPIO13)</td><td>MOSI</td></tr>
<tr><td>D0 (GPIO16)</td><td>NSS</td></tr>
<tr><td>GND</td><td>GND</td></tr>
<tr><td>3.3V</td><td>VCC</td></tr>
</table>

## Installation instructions
1. Get the firmware
2. Flash the firmware to the ESP8266
3. Reboot ESP8266
4. Upload the .LUA files to the ESP8266
5. Connect to ESP8266 and set config.

The LoRaWanGateway needs quite some RAM and processing power, so it is necessary to flash firmware that uses 
as little resources as possible. The firmware that we will use contains only the modules needed.
There are two ways to obtain the minimum build:
+ Build and download the latest NodeMCU firmware on https://nodemcu-build.com/

	OR

+ Use the build that I supply in the firmware directory, along with the NodeMCU flasher application.

### Steps to follow
#### 1. Get the firmware
##### Method A: Use the firmware that is supplied in this repository
In the directory "firmware" look for the .bin file that starts with "nodemcu-master"
##### Method B: Build your own firmware
+ Get the latest NodeMCU firmware on https://nodemcu-build.com/
	+ select the master branch
	+ select the following modules: bit, CJSON, encoder, file, GPIO, net, node, RTC time, SNTP, SPI, timer, UART, WiFi
#### 2. Flash the firmware to the ESP8266
##### Method A: Windows only, use the GUI tool supplied in this repository
+ connect your ESP8266 to a serial port
+ start ESP8266Flasher.exe in the firmware directory
+ choose the correct serial port
+ push the Flash button
##### Method B: Use the command line tool esptool.py
A Python-based, open source, platform independent, utility.  
https://nodemcu.readthedocs.io/en/master/en/flash/#esptoolpy
##### Method C: Use the python GUI tool NodeMCU PyFlasher
Self-contained NodeMCU flasher with GUI based on esptool.py and wxPython.  
https://nodemcu.readthedocs.io/en/master/en/flash/#nodemcu-pyflasher
#### 3. Reboot the ESP8266
#### 4. Upload the .lua files to the ESP8266
Suggested tool: nodemcu-uploader.py  
https://github.com/kmpm/nodemcu-uploader  
+ Upload all files in the src directory to your ESP8266
+ Restart your ESP8266
+ The LoRaWanGateway will start after first compiling all your sources
#### 5. Connect to the ESP8266 and set the config
Use a serial connection at 115200 baud to connect and configure your gateway:
+ Register your wifi network
	+ in the Lua shell run 
		+ wifi.setmode(wifi.STATION)
		-- connect to Access Point (DO save config to flash)
		+ station_cfg={}
		+ station_cfg.ssid="NODE-AABBCC"
		+ station_cfg.pwd="password"
		+ station_cfg.save=true
		+ wifi.sta.config(station_cfg)
		+ wifi.sta.autoconnect(1)
	+ Your ESP8266 will remember your wifi settings!
	

### Configuration
The LoRaWanGateway is configured to listen for all spreadingsfactors on EU channel 0 (868.1 Mhz).

The LoRaWanGateway can be run in two modes
+ Listen to all SF's, signal detection by RSSI
+ Listen to a single SF, lora signal detection

When listening to a single SF, the range of your gateway will increase a lot because messages below the noise 
floor will be received too.

Changing the configuration can be done from the LUA shell using the new CONFIG object.
Values can be changed using CONFIG.*PARAMETER*=*VALUE* i.e. CONFIG.GW_FREQ=902300000 for listening to channel 
0 on the US915 band.  
CONFIG.print() shows the current configuration.  
CONFIG.save() saves the current configuration.  
<table>
<tr><th>Parameter</th><th>Description</th><th>Default</th></tr>
<tr><td>GW_HOSTNAME</td><td>Hostname for telnet server</td><td>"lorawangw"</td></tr>
<tr><td>GW_NTP_SERVER</td><td>Dns name of the ntp server to connect</td><td>"nl.pool.ntp.org"</td></tr>
<tr><td>GW_ROUTER</td><td>Dns name of the router to connect</td><td>"router.eu.thethings.network"</td></tr>
<tr><td>GW_PORT</td><td>Port number of the router to connect</td><td>1700</td></tr>
<tr><td>GW_FREQ</td><td>Frequency (in Hz) to listen to</td><td>868100000</td></tr>
<tr><td>GW_BW</td><td>BW to listen to</td><td>"BW125"</td></tr>
<tr><td>GW_SF</td><td>SF to listen to ("SF7".."SF12"|"ALL")</td><td>"ALL" (listen to all SF's)</td></tr>
<tr><td>GW_ALT</td><td>Altitude of your gateway location</td><td>0</td></tr>
<tr><td>GW_LAT</td><td>Latitude of your gateway location</td><td>"0.0"</td></tr>
<tr><td>GW_LON</td><td>Longitude of your gateway location</td><td>"0.0"</td></tr>
<tr><td>GW_NSS</td><td>NSS pin number</td><td>0</td></tr>
<tr><td>GW_DIO0</td><td>DIO0 pin number</td><td>1</td></tr>
<tr><td>GW_DIO1</td><td>DIO1 pin number</td><td>2</td></tr>
</table>

The gateway will listen to only one specific channel. It will send on whatever channel or datarate the router 
seems fit...

## Remote access using telnet

You can connect to your gateway using a telnet client. If you connect to *GW_HOSTNAME*:23 you will get access 
to the LUA shell of your gateway.  
From this shell you can monitor your gateway, and execute lua commands like node.restart() and =node.heap().

## Statistics

You can view statistics by entering statistics() in the LUA shell or in your telnet terminal.

## Revisions

* 2017-04-19 restored compatability with nodemcu dev branch, rewrote deprications (thanx to @xenek [issue #30](https://github.com/JaapBraam/LoRaWanGateway/issues/30) )
* 2017-04-18 add GW_NTP_SERVER [issue #29](https://github.com/JaapBraam/LoRaWanGateway/issues/29)
* 2017-04-18 updated documentation (build master firmware) and updated packeged firmware
* 2017-04-18 make router portnumber configurable (thanx to @dlarue [issue #25](https://github.com/JaapBraam/LoRaWanGateway/issues/25) )
* 2017-04-18 fix bug parsing frequency in txpk packets (thanx to @dlarue [issue #26](https://github.com/JaapBraam/LoRaWanGateway/issues/26))
* 2017-03-21 fix message size limit (thanx to @joscha [issue #22](https://github.com/JaapBraam/LoRaWanGateway/issues/22) )
* 2017-02-03 increase margin on transmit timing (see [issue #16](https://github.com/JaapBraam/LoRaWanGateway/issues/16))
* 2017-01-31 added statistics, formatted CONFIG.print()
* 2017-01-30 changed configuration:added CONFIG object, telnet support, refactor to use local variables
* 2017-01-26 more accurate time in rxpk message
* 2017-01-16 support for listening on a single SF, drastically increasing range
* 2017-01-15 add documentation [from the TTN forum](https://www.thethingsnetwork.org/forum/t/single-channel-gateway/798/227) and [issue #10](https://github.com/JaapBraam/LoRaWanGateway/issues/10)
* 2017-01-15 fix for initialization using GW_NSS parameter
* 2017-01-15 add firmware directory containing flasher and nodemcu firmware
+ 2017-01-08 change UDP send because latest firmware changed udpsocket.send method.
+ 2016-09-21 add GW_ALT and GW_NSS parameters to init.lua, fix stat message
+ 2016-08-12 measure RSSI in Lora mode, speed up SPI bus, cpufreq 160Mhz
+ 2016-08-08 receive messages on all spreading factors
+ 2016-07-04 changed SX1278 NSS pin to D0
+ 2016-07-03 refactor to use integer version of firmware in order to have more free resources
+ 2016-07-02 initial revision


 



