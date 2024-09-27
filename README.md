[![Platform Build](https://github.com/sle118/squeezelite-esp32/actions/workflows/Platform_build.yml/badge.svg)](https://github.com/sle118/squeezelite-esp32/actions/workflows/Platform_build.yml)
# Squeezelite-esp32
## Forewords
**More and more people seems to use this without a LMS server, just for BT, AirPlay or Spotify. It's fine but understand that squeezeliteESP32 is primarily a Logitech Media Server player and has been designed around that concept. All the others are add-ons stitched to it, so other modes have their shortcomings. So please make sure you read [this](#Additional-configuration-notes-from-the-Web-UI) before opening an issue**
## What is this?
Squeezelite-esp32 is an audio software suite made to run on espressif's esp32 and esp32-s3 wifi (b/g/n) and bluetooth chipsets. It offers the following capabilities

- Stream your local music and connect to all major on-line music providers (Spotify, Deezer, Tidal, Qobuz) using [Logitech Media Server - a.k.a LMS](https://forums.slimdevices.com/) and enjoy multi-room audio synchronization. LMS can be extended by numerous plugins and can be controlled using a Web browser or dedicated applications (iPhone, Android). It can also send audio to UPnP, Sonos, ChromeCast and AirPlay speakers/devices.
- Stream from a **Bluetooth** device (iPhone, Android)
- Stream from an **AirPlay** controller (iPhone, iTunes ...) and enjoy synchronization multiroom as well (although it's AirPlay 1 only)
- Stream directly from **Spotify** using SpotifyConnect (thanks to [cspot](https://github.com/feelfreelinux/cspot)) - please read carefully [this](#spotify)

Depending on the hardware connected to the esp32, you can send audio to a local DAC, to SPDIF or to a Bluetooth speaker. The bare minimum required hardware is a WROVER module with 4MB of Flash and 4MB of PSRAM (https://www.espressif.com/en/products/modules/esp32). With that module standalone, just apply power and you can stream to a Bluetooth speaker. You can also send audio to most I2S DAC as well as to SPDIF receivers using just a cable or an optical transducer.

But squeezelite-esp32 is highly extensible and you can add

- [Buttons](#buttons) and [Rotary Encoder](#rotary-encoder) and map/combine them to various functions (play, pause, volume, next ...)
- [GPIO expander](#gpio-expanders) (buttons, led and rotary)
- [IR receiver](#infrared) (no pullup resistor or capacitor needed, just the 38kHz receiver)
- [Monochrome, GrayScale or Color displays](#display) using SPI or I2C (supported drivers are SH1106, SSD1306, SSD1322, SSD1326/7, SSD1351, ST7735, ST7789 and ILI9341).
- [LED strip](#led-strip) for VU-meter 
- [Ethernet](#ethernet) using a Microchip LAN8720 with RMII interface or Davicom DM9051/W5500 over SPI.

Other features include

 - Resampling (16 bits mode)
 - 10-bands equalizer (16 bits mode)
 - Automatic initial setup using any WiFi device 
 - Full web interface for further configuration/management
 - Firmware over-the-air update 

To control the equalizer or use the display on LMS, a new player model is required and this is provided through a plugin that is part of LMS' 3rd party repositories

## Performances 
*(opinions presented here so I = @philippe44)*
The main build of squeezelite-esp32 is a 16 bits internal core with all calculations in 32 bits or float precision. This is a design choice I've made to preserve CPU performances (it is already stretching a lot the esp32 chipset) and optimize memory usage as we only have 4MB of usable RAM. Some might correctly comment that the WROVER module have 8MB of RAM, but the processor is only able to address 4MB and the remaining 4MB must be paginated by smaller blocks and I don't have patience to that. 

Now, when I did the porting of squeezelite to esp32, I've also made the core 16 or 32 bits compatible at compile-time. So far, it works in 32 bits but less tests have been done. You can chose to compile it in 32 bits mode. I'm not very interested above 16 bits samples because it does not bring anything (I have an engineering background in theory of information). 

| Capability                 |16 bits|32 bits| comment         						          |	
|----------------------------|-------|-------|--------------------------------------------------------------------|
| max sampling rate          |  192k |  96k  | 192k is very challenging, especially when combined with display    |
| max bit depth              |  16   |  24   | 24 bits are truncated in 16 bits mode                              | 
| spdif			     |16 bits|20 bits|		                                                          |
| mp3, aac, opus, ogg/vorbis |  48k  |  48k  |		                                                          |
| alac, flac, ogg/flac       |  96k  |  96k  | 		                                                          |
| pcm, wav, aif              |  192k |  96k  |		                                                          |
| equalizer                  |   Y   |   N   | 48kHz max (after resampling) - equalization skipped on >48k tracks |
| resampling                 |   Y   |   N   |		                                                          |
| cross-fade                 |  10s  |  <5s  | depends on buffer size and sampling rate		                  |

The esp32 must run at 240 MHz, with Quad-SPI I/O at 80 MHz and a clock of 40 Mhz. Still, it's a lot to run, especially knowing that it has a serial Flash and PSRAM, so kudos to Espressif for their chipset optimization. Now, to have all the decoding, resampling, equalizing, gain, display, spectrum/vu is a very (very) delicate equilibrium between use of internal /external RAM, tasks priorities and buffer handling. It is not perfect and the more you push the system to the limit, the higher the risk that some files would not play (see below). In general, the display will always have the lowest priority and you'll notice slowdown in scrolling and VU/Spectrum refresh rates. Now, even display thread has some critical section and impacts the capabilities. For example, a 16 bits-depth color display with low SPI speed might prevent 24/96 flac to work but still work with pcm 24/96

In 16 bits mode, although 192 kHz is reported as max rate, it's highly recommended to limit reported sampling rate to 96k (-Z 96000). Note that some high-speed 24/96k on-line streams might stutter because of TCP/IP stack performances. It is usually due to the fact that the server sends small packets of data and the esp32 cannot receive encoded audio fast enough, regardless of task priority settings (I've tried to tweak that a fair bit). The best option in that case is to let LMS proxy the stream as it will provide larger chunks and a "smoother" stream that can then be handled.

Note as well that some codecs consume more CPU than others or have not been optimized as much. I've done my best to tweak these, but that level of optimization includes writing some assembly which is painful. One very demanding codec is AAC when files are encoded with SBR. It allows reconstruction of upper part of spectrum and thus higher sampling rate, but the codec spec is such that this is optional, you can decode simply lower band and accept lower sampling rate - See the AAC_DISABLE_SBR option below.

**IMPORTANT: on esp32 (not esp32-s3), using Spotify with SPDIF produces stuttering audio when "stats" are enabled. You MUST disable them**

## Supported Hardware
Any esp32-based hardware with at least 4MB of flash and 4MB of PSRAM will be capable of running squeezelite-esp32 and there are various boards that include such chip. A few are mentionned below, but any should work. You can find various help & instructions [here](https://forums.slimdevices.com/showthread.php?112697-ANNOUNCE-Squeezelite-ESP32-(dedicated-thread))

**For the sake of clarity, WROOM modules DO NOT work as they don't include PSRAM. Some designs might add it externally, but it's (very) unlikely.**

### Raw WROVER module
Per above description, a [WROVER module](https://www.espressif.com/en/products/modules/esp32) is enough to run Squeezelite-esp32, but that requires a bit of tinkering to extend it to have analogue audio or hardware buttons (e.g.) 

Please note that when sending to a Bluetooth speaker (source), only 44.1 kHz can be used, so you either let LMS do the resampling, but you must make sure it only sends 44.1kHz tracks or enable internal resampling (using -R) option. If you connect a DAC, choice of sample rates will depends on its capabilities. See below for more details.

Most DAC will work out-of-the-box with simply an I2S connection, but some require specific commands to be sent using I2C. See DAC option below to understand how to send these dedicated commands. There is build-in support for TAS575x, TAS5780, TAS5713 and AC101 DAC.

### Raw WROOM esp32-s3 module
The esp32-s3 based modules like [this](https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf) are also supported but requires esp-idf 4.4. It is not yet part of official releases, but it compiles & runs. The s3 does not have bluetooth audio. Note that CPU performances are greatly enhanced.

### SqueezeAMP
This is the main hardware companion of Squeezelite-esp32 and has been developped together. Details on capabilities can be found [here](https://forums.slimdevices.com/showthread.php?110926-pre-ANNOUNCE-SqueezeAMP-and-SqueezeliteESP32) and [here](https://github.com/philippe44/SqueezeAMP).

If you want to rebuild, use the `squeezelite-esp32-SqueezeAmp-sdkconfig.defaults` configuration file.

NB: You can use the pre-build binaries SqueezeAMP4MBFlash which has all the hardware I/O set properly. You can also use the generic binary I2S4MBFlash in which case the NVS parameters shall be set to get the exact same behavior
- set_GPIO: `12=green,13=red,34=jack,2=spkfault`
- bat_config: `channel=7,scale=20.24`
- dac_config: `model=TAS57xx,bck=33,ws=25,do=32,sda=27,scl=26,mute=14:0`
- spdif_config: `bck=33,ws=25,do=15`

The IR can be used as a wake-up signal using (setting `sleep_config` with `wake=0:0`). It's a pull-up so it stays at 1 when not receiving anything which means it cannot be used in conjuction with other wake-up IOs. See [Sleeping](#sleeping) for more details regarding the limitation of waking-up upon multiple inputs.

### MuseLuxe
This portable battery-powered [speaker](https://raspiaudio.com/produit/esp-muse-luxe) is compatible with squeezelite-esp32 for which there is a dedicated build supplied with every update. If you want to rebuild, use the `squeezelite-esp32-Muse-sdkconfig.defaults` configuration file.

NB: You can use the pre-build binaries Muse4MBFlash which has all the hardware I/O set properly. You can also use the generic binary I2S4MBFlash in which case the NVS parameters shall be set to get the exact same behavior
- target: `muse`
- bat_config: `channel=5,scale=7.48,atten=3,cells=1`
- spi_config: `"mosi=15,miso=2,clk=14` *(this one is probably optional)*
- dac_config: `model=I2S,bck=5,ws=25,do=26,di=35,i2c=16,sda=18,scl=23,mck=0`
- dac_controlset: `{"init":[ {"reg":0,"val":128}, {"reg":0,"val":0}, {"reg":25,"val":4}, {"reg":1,"val":80}, {"reg":2,"val":0},	{"reg":8,"val":0}, {"reg":4,"val":192},	{"reg":0,"val":18}, {"reg":1,"val":0}, {"reg":23,"val":24}, {"reg":24,"val":2}, {"reg":38,"val":9}, {"reg":39,"val":144}, {"reg":42,"val":144}, {"reg":43,"val":128}, {"reg":45,"val":128}, {"reg":27,"val":0}, {"reg":26,"val":0}, {"reg":2,"val":240}, {"reg":2,"val":0},	{"reg":29,"val":28}, {"reg":4,"val":48}, {"reg":25,"val":0}, {"reg":46,"val":33}, {"reg":47,"val":33} ]}`
- actrls_config: buttons
- define a "buttons" variable with: `[{"gpio":32, "pull":true, "debounce":10, "normal":{"pressed":"ACTRLS_VOLDOWN"}}, {"gpio":19, "pull":true, "debounce":40, "normal":{"pressed":"ACTRLS_VOLUP"}}, {"gpio":12, "pull":true, "debounce":40, "long_press":1000, "normal":{"pressed":"ACTRLS_TOGGLE"},"longpress":{"pressed":"ACTRLS_POWER"}}]`

### ESP32-A1S
Works with [ESP32-A1S](https://docs.ai-thinker.com/esp32-a1s) module that includes audio codec and headset output. You still need to use a demo board like [this](https://aliexpress.com/item/4000130915903.html) or an external amplifier if you want direct speaker connection. Note that there is a version with AC101 codec and another one with ES8388 with probably two variants - these boards are a mess (see below)

The board shown above has the following IO set
- amplifier: GPIO21
- key2: GPIO13, key3: GPIO19, key4: GPIO23, key5: GPIO18, key6: GPIO5 (to be confirmed with dip switches)
- key1: not sure, using GPIO36 in a matrix
- jack insertion: GPIO39 (inserted low)
- D4 -> GPIO22 used for green LED (active low)
- D5 -> GPIO19 (muxed with key3)
- The IO connector also brings GPIO5, GPIO18, GPIO19, GPIO21, GPIO22 and GPIO23 (don't forget it's muxed with keys!)
- The JTAG connector uses GPIO 12, 13, 14 and 15 (see dip switch) but these are also used for SD-card (and GPIO13 is key2 as well)
- It's always possible to re-use GPIOO (download at boot) and GPIO1/GPIO3 which are RX/TX of UART0 but you'll lose trace

(note that some GPIO need pullups)

So a possible config would be
- set_GPIO: `21=amp,22=green:0,39=jack:0`
- a button mapping: 
	```json
	[{"gpio":5,"normal":{"pressed":"ACTRLS_TOGGLE"}},{"gpio":18,"pull":true,"shifter_gpio":5,"normal":{"pressed":"ACTRLS_VOLUP"}, "shifted":{"pressed":"ACTRLS_NEXT"}}, {"gpio":23,"pull":true,"shifter_gpio":5,"normal":{"pressed":"ACTRLS_VOLDOWN"},"shifted":{"pressed":"ACTRLS_PREV"}}]
	```
for **AC101**
- dac_config: `model=AC101,bck=27,ws=26,do=25,di=35,sda=33,scl=32`
 
for **ES8388** (it seems that there are variants with same version number - a total mess)
- dac_config: `model=ES8388,bck=5,ws=25,do=26,sda=18,scl=23,i2c=16`
or
- dac_config: `model=ES8388,bck=27,ws=25,do=26,sda=33,scl=32,i2c=16`

### T-WATCH2020 by LilyGo
This is a fun [smartwatch](http://www.lilygo.cn/prod_view.aspx?TypeId=50036&Id=1290&FId=t3:50036:3) based on ESP32. It has a 240x240 ST7789 screen and onboard audio. Not very useful to listen to anything but it works. This is an example of a device that requires an I2C set of commands for its DAC/APU (see below). There is a build-option if you decide to rebuild everything by yourself, otherwise the I2S default option works with the following parameters

- dac_config: `model=I2S,bck=26,ws=25,do=33,i2c=53,sda=21,scl=22`
- dac_controlset:
	```json
	{ "init": [ {"reg":41, "val":128}, {"reg":18, "val":255} ], "poweron": [ {"reg":18, "val":64, "mode":"or"} ], "poweroff": [ {"reg":18, "val":191, "mode":"and"} ] }
	```
- spi_config: `dc=27,data=19,clk=18`
- display_config: `SPI,driver=ST7789,width=240,height=240,cs=5,back=12,speed=16000000,HFlip,VFlip`

### ESP32-WROVER + I2S DAC
Squeezelite-esp32 requires esp32 chipset and 4MB PSRAM. ESP32-WROVER meets these requirements. To get an audio output an I2S DAC can be used. Cheap PCM5102 I2S DACs work but many others also do. PCM5012 DACs can be hooked up via:

I2S - WROVER  
VCC - 3.3V  
3.3V - 3.3V  
GND - GND  
FLT - GND  
DMP - GND  
SCL - GND  
BCK - (BCK - see below)  
DIN - (DO - see below)  
LCK - (WS - see below)
FMT - GND  
XMT - 3.3V 

Use the `squeezelite-esp32-I2S-4MFlash-sdkconfig.defaults` configuration file.

### SqueezeAmpToo !

And the super cool project https://github.com/rochuck/squeeze-amp-too

## Configuration
To access NVS, in the webUI, go to credits and select "shows nvs editor". Go into the NVS editor tab to change NFS parameters. In syntax description below \<\> means a value while \[\] describe optional parameters. 

As mentionned above, there are a few dedicated builds that are provided today: SqueezeAMP and Muse but if you build it yourself, you can also create a build for T-WATCH2020. The default build is a generic firmware named I2S which can be configured through NVS to produce *exactly* the same results than dedicated builds. The difference is that parameters must be entered and can accidently be erased. The GUI provides a great help to load "known config sets" as well. 

By design choice, there is no code that is only embedded for a given version, all code is always there. The philosophy is to minimize as much as possible platform-specific code and use of specific `#ifdef` is prohibited, no matter what. So if you want to add your own platfrom, please look **very hard** at the `main\KConfig.projbuild` to see how you can, using parameters below, make your device purely a configuration-based solution. When there is really no other option, look at `targets\<target>` to add your own code. I will not accept PR for code that can avoid creating such dedicated code whenever possible. The NVS "target" will be used to call target-specific code then, but again this is purely runtime, not compile-time.

### I2C
The NVS parameter "i2c_config" set the i2c's gpio used for generic purpose (e.g. display). Leave it blank to disable I2C usage. Note that on SqueezeAMP, port must be 1. Default speed is 400000 but some display can do up to 800000 or more. Syntax is
```
sda=<gpio>,scl=<gpio>[,port=0|1][,speed=<speed>]
```
**Please note that you can not use the same GPIO or port as the DAC.**

### SPI
The esp32 has 4 SPI sub-systems, one is unaccessible so numbering is 0..2 and SPI0 is reserved for Flash/PSRAM. The NVS parameter "spi_config" set the spi's gpio used for generic purpose (e.g. display). Leave it blank to disable SPI usage. The DC parameter is needed for displays. Syntax is
```
data|mosi=<gpio>,clk=<gpio>[,dc=<gpio>][,host=1|2][,miso=<gpio>]
``` 
Default and only "host" is 1 as others are used already by flash and spiram. The optional "miso" (MasterInSlaveOut) parameter is only used when SPI bus is bi-directional and shared with other peripheral like ethernet, gpio expander. Note that "data" can also be named "mosi" (MasterOutSlaveIn). 
### DAC/I2S
The NVS parameter "dac_config" set the gpio used for i2s communication with your DAC. You can define the defaults at compile time but nvs parameter takes precedence except for named configurations
```
bck=<gpio>,ws=<gpio>,do=<gpio>[,mck=0|1|2][,mute=<gpio>[:0|1][,model=TAS57xx|TAS5713|AC101|WM8978|ES8388|I2S][,sda=<gpio>,scl=<gpio>[,i2c=<addr>]]
```
if "model" is not set or is not recognized, then default "I2S" is used. The option "mck" is used for some codecs that require a master clock (although they should not). By default GPIO0 is used as MCLK and only recent builds (post mid-2023) can use 1 or 2. Also be aware that this cannot coexit with RMII Ethernet (see ethernet section below). I2C parameters are optional and only needed if your DAC requires an I2C control (See 'dac_controlset' below). Note that "i2c" parameters are decimal, hex notation is not allowed.

So far, TAS57xx, TAS5713, AC101, WM8978 and ES8388 are recognized models where the proper init sequence/volume/power controls are sent. For other codecs that might require an I2C commands, please use the parameter "dac_controlset" that allows definition of simple commands to be sent over i2c for init, power, speaker and headset on and off using a JSON syntax:
```json
{ <command>: [ <item1>, <item2>, ... <item3> ],
  <command>: [ <item1>, <item2>, ... <item3> ],
  ... }
```
Where `<command>` is one of init, poweron, poweroff, speakeron, speakeroff, headseton, headsetoff (it **must** be an array even for a single item). Item is any of the following elements
```
{"reg":<register>,"val":<value>,"mode":<nothing>|"or"|"and"}
{"gpio":<gpio>,"level":0|1}
{"delay":<ms>}
```
This is standard JSON notation, so if you are not familiar with it, Google is your best friend. Be aware that the '...' means you can have as many entries as you want, it's not part of the syntax. Every section is optional, but it does not make sense to set i2c in the 'dac_config' parameter and not setting anything here. 

The `reg` key allow to write registers on i2c bus. The parameter `mode` allows to *or* the register with the value or to *and* it. Don't set `mode` if you simply want to write. The `val` parameter can be an array [v1, v2,...] to write a serie of bytes in a single i2c burst (in that case 'mode' is ignored). **Note that all values must be decimal**. You can use a validator like [this](https://jsonlint.com) to verify your syntax. The `gpio` key is simply to set a gpio as part of DAC action and `delay` allows a pause between elements.

The 'power' command is used when powering on/off the DAC after the idle period (see -C option of squeezelite) and the 'speaker/headset' commands are sent when switching between speakers and headsets (see headset jack detection).

NB: For named configurations ((SqueezeAMP, Muse ... all except I2S), all this is ignored. For know codecs, the built-in sequences can be overwritten using dac_controlset

**Please note that you can not use the same GPIO or port as the I2C.**

### SPDIF
The NVS parameter "spdif_config" sets the i2s's gpio needed for SPDIF. 

SPDIF is made available by re-using i2s interface in a non-standard way, so although only one pin (DO) is needed, the controller must be fully initialized, so the bit clock (bck) and word clock (ws) must be set as well. As i2s and SPDIF are mutually exclusive, you can reuse the same IO if your hardware allows so.

You can define the defaults at compile time but nvs parameter takes precedence except for named configurations (SqueezeAMP, Muse ...)

Leave it blank to disable SPDIF usage, you can also define them at compile time using "make menuconfig". Syntax is 
```
bck=<gpio>,ws=<gpio>,do=<gpio>
```
NB: For named configurations, this is ignored

The maximum bit depth is 24 bits, even in 32 bits mode (this a SPDIF limitation - thank @UrbanLienert for theupdate from 20 to 24 bit). Now, you can also get SPDIF using a specialized chip that offers a I2S interface like a DAC but spits out SPDIF (optical and coax). Refers to DAC chapter then.

If you want coax, you can also use a poor-man's trick to generate signal from a 3.3V GPIO. All that does is dividing the 3.3V to generate a 0.6V peak-to-peak and then remove DC
```
                          100nF
GPIO  ----210ohm-----------||---- coax S/PDIF signal out
                    |
                  110ohm
                    |
Ground -------------------------- coax signal ground
```

### Display
The NVS parameter "display_config" sets the parameters for an optional display. It can be I2C (see [here](#i2c) for shared bus) or SPI (see [here](#spi) for shared bus) Syntax is
```
I2C,width=<pixels>,height=<pixels>[address=<i2c_address>][,reset=<gpio>][,HFlip][,VFlip][driver=SSD1306|SSD1326[:1|4]|SSD1327|SH1106]
SPI,width=<pixels>,height=<pixels>,cs=<gpio>[,back=<gpio>][,reset=<gpio>][,speed=<speed>][,HFlip][,VFlip][driver=SSD1306|SSD1322|SSD1326[:1|4]|SSD1327|SH1106|SSD1675|ST7735|ST7789[:x=<offset>][:y=<offset>]|ILI9341[:16|18][,rotate]]
```
- back: a LED backlight used by some older devices (ST7735). It is PWM controlled for brightness
- reset: some display have a reset pin that is should normally be pulled up if unused. Most displays require reset and will not initialize well otherwise.
- VFlip and HFlip are optional can be used to change display orientation
- rotate: for non-square *drivers*, move to portrait mode. Note that *width* and *height* must be inverted then
- Default speed is 8000000 (8MHz) but SPI can work up to 26MHz or even 40MHz
- SH1106 is 128x64 monochrome I2C/SPI [here](https://www.waveshare.com/wiki/1.3inch_OLED_HAT)
- SSD1306 is 128x32 monochrome I2C/SPI [here](https://www.buydisplay.com/i2c-blue-0-91-inch-oled-display-module-128x32-arduino-raspberry-pi)
- SSD1322 is 256x64 grayscale 16-levels SPI in multiple sizes [here](https://www.buydisplay.com/oled-display/oled-display-module?resolution=159) - it is very nice
- SSD1326 is 256x32 monochrome or grayscale 16-levels SPI [here](https://www.aliexpress.com/item/32833603664.html?spm=a2g0o.productlist.0.0.2d19776cyQvsBi&algo_pvid=c7a3db92-e019-4095-8a28-dfdf0a087f98&algo_expid=c7a3db92-e019-4095-8a28-dfdf0a087f98-1&btsid=0ab6f81e15955375483301352e4208&ws_ab_test=searchweb0_0,searchweb201602_,searchweb201603_)
- SSD1327 is 128x128 16-level grayscale SPI [here](https://www.amazon.com/gp/product/B079N1LLG8/ref=ox_sc_act_title_1?smid=A1N6DLY3NQK2VM&psc=1) - artwork can be up to 96x96 with vertical vu-meter/spectrum
- SSD1351 is 128x128 65k/262k color SPI [here](https://www.waveshare.com/product/displays/lcd-oled/lcd-oled-3/1.5inch-rgb-oled-module.htm)
- SSD1675 is an e-ink paper and is experimental as e-ink is really not suitable for LMS du to its very low refresh rate
- ST7735 is a 128x160 65k color SPI [here](https://www.waveshare.com/product/displays/lcd-oled/lcd-oled-3/1.8inch-lcd-module.htm). This needs a backlight control. Some have X/Y offsets betwen the driver and the glass (green/black/red models) that can be added using "x" and "y" options (case sensitive!)
- ST7789 is a 240x320 65k (262k not enabled) color SPI [here](https://www.waveshare.com/product/displays/lcd-oled/lcd-oled-3/2inch-lcd-module.htm). It also exist with 240x240 displays. See **rotate** for use in portrait mode
- ILI9341 is another 240x320 65k (262k capable) color SPI. I've not used it much, the driver it has been provided by one external contributor to the project

You can tweak how the vu-meter and spectrum analyzer are displayed, as well as size of artwork through a dedicated menu in player's settings (don't forget to add the plugin).

The NVS parameter "metadata_config" sets how metadata is displayed for AirPlay and Bluetooth. Syntax is
```
[format=<display_content>][,speed=<speed>][,pause=<pause>][,artwork[:0|1]]
```
- 'speed' is the scrolling speed in ms (default is 33ms)
- 'pause' is the pause time between scrolls in ms (default is 3600ms)
- 'format' can contain free text and any of the 3 keywords `%artist%`, `%album%`, `%title%`. Using that format string, the keywords are replaced by their value to build the string to be displayed. Note that the plain text following a keyword that happens to be empty during playback of a track will be removed. For example, if you have set format=`%artist% - %title%` and there is no artist in the metadata then only `<title>` will be displayed not ` - <title>`.
- 'artwork' enables coverart display, if available (does not work for Bluetooth). The optional parameter indicates if the artwork should be resized (1) to fit the available space. Note that the built-in resizer can only do 2,4 and 8 downsizing, so fit is not optimal. The artwork will be placed at the right of the display for landscape displays and underneath the two information lines for others (there is no user option to tweak that).

### Infrared
You can use any IR receiver compatible with NEC protocol (38KHz) or RC5. Vcc, GND and output are the only pins that need to be connected, no pullup, no filtering capacitor, it's a straight connection.

The IR codes are send "as is" to LMS, so only a Logitech SB remote from Boom, Classic or Touch will work. I think the file Slim_Devices_Remote.ir in the "server" directory of LMS can be modified to adapt to other codes, but I've not tried that.

In AirPlay and Bluetooth mode, only these native remotes are supported, I've not added the option to make your own mapping

See "set GPIO" below to set the GPIO associated to infrared receiver (option "ir"). 

### Set GPIO
The parameter "set_GPIO" is used to assign GPIO to various functions.

GPIO can be set to GND provide or Vcc at boot. This is convenient to power devices that consume less than 40mA from the side connector. Be careful because there is no conflict checks being made wrt which GPIO you're changing, so you might damage your board or create a conflict here. 

The `<amp>` parameter can use used to assign a GPIO that will be set to active level (default 1) when playback starts. It will be reset when squeezelite becomes idle. The idle timeout is set on the squeezelite command line through `-C <timeout>`

The `<power>` parameter can use used to assign a GPIO that will be set to active level (default 1) when player is powered on and reset when powered off (in LMS, does not apply to AirPlay, Spotify or BT).

If you have an audio jack that supports insertion (use :0 or :1 to set the level when inserted), you can specify which GPIO it's connected to. Using the parameter jack_mutes_amp allows to mute the amp when headset (e.g.) is inserted.

You can set the Green and Red status led as well with their respective active state (:0 or :1) or specific the chipset if you use addressable RGB led.

The `<ir>` parameter set the GPIO associated to an IR receiver. No need to add pullup or capacitor

Syntax is:

```
<gpio>=Vcc|GND|amp[:1|0]|power[:1:0]|ir[:nec|rc5]|jack[:0|1]|green[:0|1|ws2812]|red[:0|1|ws2812]|spkfault[:0|1][,<repeated sequence for next GPIO>]
```
You can define the defaults for jack, spkfault leds at compile time but nvs parameter takes precedence except for named configurations ((SqueezeAMP, Muse ...) where these are forced at runtime.
**Note that gpio 36 and 39 are input only and cannot use interrupt. When set to jack or speaker fault, a 100ms polling checks their value but that's expensive**
 
### GPIO expanders
It is possible to add GPIO expanders using I2C or SPI bus. They should mainly be used for buttons but they can support generic-purpose outputs as well. These additional GPIOs can be numbered starting from an arbitrary value (40 and above as esp32 has GPIO 0..39). Then these new "virtual" GPIOs from (e.g) 100 to 115 can be used in [button](#Buttons) configuration, [set_GPIO](#set-gpio) or other config settings.

Each expander can support up to 32 GPIO. To use an expander for buttons, an interrupt must be provided, polling mode is not acceptable. An expander w/o interruption can still be configured, but only output will be usable. Note that the same interrupt can be shared accross expanders, as long as they are using open drain or open collectors (which they probably all do)

The parameter "gpio_exp_config" is a semicolon (;) separated list with following syntax for each expander
```
model=<model>,addr=<addr>,[,port=system|dac][,base=<n>][,count=<n>][,intr=<gpio>][,cs=<gpio>][,speed=<Hz>]
```	
- model: pca9535, pca85xx, mcp23017, aw9523 and mcp23s17 (SPI version)
- addr: chip i2c/spi address (decimal)
- port (I2C): use either "system" port (shared with display for example) or "dac" port (system is default)
- cs (SPI): gpio used for Chip Select
- speed (SPI): speed of the SPI bus for that device (in Hz)	
- base: GPIO numbering offset to use everywhere else (default 40 on esp32 and 48 on esp32-s3)
- count: number of GPIO of expander (default 16 - might be obsolted if model if sufficient to decide)
- intr: real GPIO to use as interrupt.
	
Note that PWM ("led_brightness" below) is not supported for expanded GPIOs and they cannot be used for high speed or precise timing signals like CS, D/C, Reset and Ready. Buttons, rotary encoder, amplifier control and power are supported. Depending on the actual chipset, pullup or pulldown might be supported so you might have to add external resistors (only MCP23x17 does pullup). The pca8575 is not a great chip, it generate a fair bit of spurious interrupts when used for GPIO out. When using a SPI expander, the bus must be configured using shared [SPI](#SPI) bus

### LED 
See [set_GPIO](#set-gpio) for how to set the green and red LEDs (including addressable RGB ones). In addition, their brightness can be controlled using the "led_brigthness" parameter. The syntax is
```
[green=0..100][,red=0..100]
```
NB: For named configuration, GPIO affected to green and red LED cannot be changed but brightness option applies

### LED Strip
One LED strip with up to 255 addressable LEDs can be configured to offer enhanced visualizations.  The VU Meter visualizer includes a battery status indicator (see Battery).  Currently only WS2812B LEDs are supported.  Set the LED Strip hardware configuration, or the NVS led_vu_config syntax is 
```
type=[WS2812],length=<n>,gpio=<dataPin>[,scale=<gain>]
```
where `<n>` is the number of LEDs in the strip (1..255).  A `<scale>` gain value (percentage) can be added to enhance effect responses. 

The latest LMS plugin update is required to set the visualizer mode and brightness in the ESP32 Settings page for the player, or a controllable display (see Extra/SqueezeESP32 menus).  The plugin adds additional LMS CLI commands.

| Command                                            | Notes       |
| -------------------------------------------------- | ----------- |
| \<playerid\> led_visual \[\<mode\>\] \[\<brightness\>\] | Toggles or selects the visualizer "mode".<br />The visualizer brightness(0..255) can be controlled using the "brightness" tag. |
| \<playerid\> dmx \<R,G,B,R,G,B, ... R,G,B\> \[\<offset\>\] | Sets the LED color starting at position "offset"<br />  with "R"(red),"G"(green),and "B"(blue) color sequences.<br />Add additional RGB values to the delimited string to set multiple LEDs.<br /> |

### Rotary Encoder
One rotary encoder is supported, quadrature shift with press. Such encoders usually have 2 pins for encoders (A and B), and common C that must be set to ground and an optional SW pin for press. A, B and SW must be pulled up, so automatic pull-up is provided by ESP32, but you can add your own resistors. A bit of filtering on A and B (~470nF) helps for debouncing which is not made by software. 

Encoder is normally hard-coded to respectively knob left, right and push on LMS and to volume down/up/play toggle on BT and AirPlay. Using the option 'volume' makes it hard-coded to volume down/up/play toggle all the time (even in LMS). The option 'longpress' allows an alternate mode when SW is long-pressed. In that mode, left is previous, right is next and press is toggle. Every long press on SW alternates between modes (the main mode actual behavior depends on 'volume').

There is also the possibility to use 'knobonly' option (exclusive with 'volume' and 'longpress'). This mode attempts to offer a single knob full navigation which is a bit contorded due to LMS UI's principles. Left, Right and Press obey to LMS's navigation rules and especially Press always goes to lower submenu item, even when navigating in the Music Library. That causes a challenge as there is no 'Play', 'Back' or 'Pause' button. Workaround are as of below:
- longpress is 'Play'
- double press is 'Back' (Left in LMS's terminology). 
- a quick left-right movement on the encoder is 'Pause' 

The speed of double click (or left-right) can be set using the optional parameter of 'knobonly'. This is not a perfect solution, and other ideas are welcome. Be aware that the longer you set double click speed, the less responsive the interface will be. The reason is that I need to wait for that delay before deciding if it's a single or double click. It can also make menu navigation "hesitations" being easily interpreted as 'Pause'

Use parameter rotary_config with the following syntax:

```
A=<gpio>,B=<gpio>[,SW=gpio>[[,knobonly[=<ms>]]|[[,volume][,longpress]]]]
```

HW note: all gpio used for rotary have internal pull-up so normally there is no need to provide Vcc to the encoder. Nevertheless if the encoder board you're using also has its own pull-up that are stronger than ESP32's ones (which is likely the case), then there will be crosstalk between gpio, so you must bring Vcc. Look at your board schematic and you'll understand that these board pull-up create a "winning" pull-down when any other pin is grounded. 

The SW gpio is optional, you can re-affect it to a pure button if you prefer but the volume, longpress and knobonly options make little sense as the missing switch plays an important role in these modes. You could still have the "volume" mode, but you won't be able to use it for *anything* expect volume up and down. So be aware that the use of syntax [] is a bit misleading hereabove.

See also the "IMPORTANT NOTE" on the "Buttons" section and remember that when 'lms_ctrls_raw' (see below) is activated, none of these knobonly,volume,longpress options apply, raw button codes (not actions) are simply sent to LMS

**Note that gpio 36 and 39 are input only and cannot use interrupt, so they cannot be set to A or B. When using them for SW, a 100ms polling is used which is expensive**

### Buttons
Buttons are described using a JSON string with the following syntax
```json
[
{"gpio":<num>,
 "type":"BUTTON_LOW | BUTTON_HIGH",
 "pull":[true|false],
 "long_press":<ms>,
 "debounce":<ms>,
 "shifter_gpio":<-1|num>,
 "normal": {"pressed":"<action>","released":"<action>"},
 "longpress": { <same> },
 "shifted": { <same> },
 "longshifted": { <same> },
 },
 { ... },
 { ... },
] 
```

Where (all parameters are optionals except gpio) 
 - "type": (BUTTON_LOW) logic level when the button is pressed 
 - "pull": (false) activate internal pull up/down
 - "long_press": (0) duration (in ms) of keypress to detect long press, 0 to disable it
 - "debounce": (0) debouncing duration in ms (0 = internal default of 50 ms)
 - "shifter_gpio": (-1) gpio number of another button that can be pressed together to create a "shift". Set to -1 to disable shifter
 - "normal": ({"pressed":"ACTRLS_NONE","released":"ACTRLS_NONE"}) action to take when a button is pressed/released (see below)
 - "longpress": action to take when a button is long-pressed/released (see above/below)
 - "shifted": action to take when a button is pressed/released and shifted (see above/below)
 - "longshifted": action to take when a button is long-pressed/released and shifted (see above/below)

Where `<action>` is either the name of another configuration to load (remap) or one amongst

```
ACTRLS_NONE, ACTRLS_POWER, ACTRLS_VOLUP, ACTRLS_VOLDOWN, ACTRLS_TOGGLE, ACTRLS_PLAY, 
ACTRLS_PAUSE, ACTRLS_STOP, ACTRLS_REW, ACTRLS_FWD, ACTRLS_PREV, ACTRLS_NEXT, 
BCTRLS_UP, BCTRLS_DOWN, BCTRLS_LEFT, BCTRLS_RIGHT, 
BCTRLS_PS1, BCTRLS_PS2, BCTRLS_PS3, BCTRLS_PS4, BCTRLS_PS5, BCTRLS_PS6, BCTRLS_PS7, BCTRLS_PS8, BCTRLS_PS9, BCTRLS_PS10,
KNOB_LEFT, KNOB_RIGHT, KNOB_PUSH,
ACTRLS_SLEEP,
```
Note that ACTRLS_SLEEP is not an actual button that can be sent to LMS, but it's a hook to activate deep sleep mode (see [Sleeping](#sleeping)).

One you've created such a string, use it to fill a new NVS parameter with any name below 16(?) characters. You can have as many of these configs as you can. Then set the config parameter "actrls_config" with the name of your default config

For example a config named "buttons" :
```json
[{"gpio":4,"type":"BUTTON_LOW","pull":true,"long_press":1000,"normal":{"pressed":"ACTRLS_VOLDOWN"},"longpress":{"pressed":"buttons_remap"}},
 {"gpio":5,"type":"BUTTON_LOW","pull":true,"shifter_gpio":4,"normal":{"pressed":"ACTRLS_VOLUP"}, "shifted":{"pressed":"ACTRLS_TOGGLE"}}]
``` 
Defines two buttons
- first on GPIO 4, active low. When pressed, it triggers a volume down command. When pressed more than 1000ms, it changes the button configuration for the one named "buttons_remap"
- second on GPIO 5, active low. When pressed it triggers a volume up command. If first button is pressed together with this button, then a play/pause toggle command is generated.

While the config named "buttons_remap"
```json
[{"gpio":4,"type":"BUTTON_LOW","pull":true,"long_press":1000,"normal":{"pressed":"BCTRLS_DOWN"},"longpress":{"pressed":"buttons"}},
 {"gpio":5,"type":"BUTTON_LOW","pull":true,"shifter_gpio":4,"normal":{"pressed":"BCTRLS_UP"}}]
``` 
Defines two buttons
- first on GPIO 4, active low. When pressed, it triggers a navigation down command. When pressed more than 1000ms, it changes the button configuration for the one described above
- second on GPIO 5, active low. When pressed it triggers a navigation up command. That button, in that configuration, has no shift option

Below is a difficult but functional 2-buttons interface for your decoding pleasure:

`actrls_config`:
```
buttons
```
`buttons`:
```json
[{"gpio":4,"type":"BUTTON_LOW","pull":true,"long_press":1000,
 "normal":{"pressed":"ACTRLS_VOLDOWN"},
 "longpress":{"pressed":"buttons_remap"}},
 {"gpio":5,"type":"BUTTON_LOW","pull":true,"long_press":1000,"shifter_gpio":4,
 "normal":{"pressed":"ACTRLS_VOLUP"}, 
 "shifted":{"pressed":"ACTRLS_TOGGLE"}, 
 "longpress":{"pressed":"ACTRLS_NEXT"}}
]
```
`buttons_remap`:
```json
[{"gpio":4,"type":"BUTTON_LOW","pull":true,"long_press":1000,
 "normal":{"pressed":"BCTRLS_DOWN"},
 "longpress":{"pressed":"buttons"}},
 {"gpio":5,"type":"BUTTON_LOW","pull":true,"long_press":1000,"shifter_gpio":4,
 "normal":{"pressed":"BCTRLS_UP"},
 "shifted":{"pressed":"BCTRLS_PUSH"},
 "longpress":{"pressed":"ACTRLS_PLAY"},
 "longshifted":{"pressed":"BCTRLS_LEFT"}}
]
```
**IMPORTANT NOTE**: LMS also supports the possibility to send 'raw' button codes. It's a bit complicated, so bear with me. Buttons can either be processed by SqueezeESP32 and mapped to a "function" like play/pause or they can be just sent to LMS as plain (raw) code and the full logic of press/release/longpress is handled by LMS, you don't have any control on that.

The benefit of the "raw" mode is that you can build a player which is as close as possible to a Boom (e.g.) but you can't use the remapping function nor longpress or shift logics to do your own mapping when you have a limited set of buttons. In 'raw' mode, all you really need to define is the mapping between the gpio and the button. As far as LMS is concerned, any other option in these JSON payloads does not matter. Now, when you use BT or AirPlay, the full JSON construct described above fully applies, so the shift, longpress, remapping options still work.

**Be aware that when using non "raw" mode, the CLI (Command Line Interface) of LMS is used and *must* be available without password**

There is no good or bad option, it's your choice. Use the NVS parameter "lms_ctrls_raw" to change that option
	
**Note that gpio 36 and 39 are input only and cannot use interrupt. When using them for a button, a 100ms polling is started which is expensive. Long press is also likely to not work very well**
### Ethernet 
Wired ethernet is supported by esp32 with various options but squeezeESP32 is only supporting a Microchip LAN8720 with a RMII interface like [this](https://www.aliexpress.com/item/32858432526.html) or SPI-ethernet bridges like Davicom DM9051 [that](https://www.amazon.com/dp/B08JLFWX9Z) or W5500 like [this](https://www.aliexpress.com/item/32312441357.html).

**Note:** Touch buttons that can be find on some board like the LyraT V4.3 are not supported currently.

#### RMII (LAN8720)	
- RMII PHY wiring is fixed and can not be changed

| GPIO   | RMII Signal | Notes        |
| ------ | ----------- | ------------ |
| GPIO21 | TX_EN       | EMAC_TX_EN   |
| GPIO19 | TX0         | EMAC_TXD0    |
| GPIO22 | TX1         | EMAC_TXD1    |
| GPIO25 | RX0         | EMAC_RXD0    |
| GPIO26 | RX1         | EMAC_RXD1    |
| GPIO27 | CRS_DV      | EMAC_RX_DRV  |
| GPIO0  | REF_CLK     | 50MHz clock  |

- SMI (Serial Management Interface) wiring is not fixed and you can change it either in the configuration or using "eth_config" parameter with the following syntax:
```
model=lan8720,mdc=<gpio>,mdio=<gpio>[,rst=<gpio>]
```
Connecting a reset pin for the LAN8720 is optional but recommended to avoid that GPIO0 (50MHz input clock) locks the esp32 in download mode at boot time.
- Clock
	
The APLL of the esp32 is required for the audio codec, so we **need** a LAN8720 that provides a 50MHz clock. That clock **must** be connected to GPIO0, there is no alternative. This means that if your DAC requires an MCLK, you need a recent build (later than mid-2023) to be able to select either GPIO 1 or 2.

#### SPI (DM9051 or W5500)
Ethernet over SPI is supported as well and requires less GPIOs but is obvsiously slower. SPI is the shared bus set with [spi_config](#spi). The "eth_config" parameter syntax becomes:
```
model=dm9051|w5500,cs=<gpio>,speed=<clk_in_Hz>,intr=<gpio>[,rst=<gpio>]
```
- To use the system SPI, shared with display (see spi_config) "host" must be set to -1. Any other value will reserve the SPI interface (careful of conflict with spi_config). The default "host" is 2 to avoid conflicting wiht default "spi_config" settings.
- When not using system SPI, "mosi" for data out, "miso" for data in and "clk" **must** be set
- The esp32 has a special I/O multiplexer for faster speed (up to 80 MHz) but that requires using specific GPIOs, which depends on SPI bus (See [here](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/spi_master.html) for more details)

| Pin Name | SPI1 | SPI2 |
| -------- | ---- | ---- |		
| CS       |  15  |  5   |
| SCLK	   |  14  |  18  |
| MISO	   |  12  |  19  |
| MOSI	   |  13  |  23  |
	
### Battery / ADC
The NVS parameter "bat_config" sets the ADC1 channel used to measure battery/DC voltage. The "atten" value attenuates the input voltage to the ADC input (the read value maintains a 0-1V rage) where: 0=no attenuation(0..800mV), 1=2.5dB attenuation(0..1.1V), 2=6dB attenuation(0..1.35V), 3=11dB attenuation(0..2.6V). Scale is a float ratio applied to every sample of the 12 bits ADC. A measure is taken every 10s and an average is made every 5 minutes (not a sliding window). Syntax is
```
channel=0..7,scale=<scale>,cells=<1..3>[,atten=<0|1|2|3>]
```
NB: Set parameter to empty to disable battery reading. For named configurations (SqueezeAMP, Muse ...), this is ignored (except for SqueezeAMP where number of cells is required)

### Sleeping
The esp32 can be put in deep sleep mode to save some power. How much really depends on the connected periperals, so best is to do your own measures. Waking-up from deep sleep is the equivalent of a reboot, but as the chip takes a few seconds to connect, it's still an efficient process.

The esp32 can enter deep sleep after an audio inactivity timeout, after a button has been pressed, after a GPIO is set to a given level (there is a subtle difference, see below) or if the battery reaches a threashold. It wakes up only on some GPIO events. Note that *all* GPIO are isolated when sleeping (unless they are set with the `rtc`option) so you can not assume anything about their value, except that they will not drain current. The `rtc` option allows to keep some GPIO (from the RTC domain only) either pulled up or down. This can be useful if you want to keep some periperal active, for example a GPIO expander whose interrupt will be used to wake-up the system.

The NVS parameter `sleep_config` is mostly used for setting sleep conditions
```
[delay=<mins>][,sleep=<gpio>[:0|1]][,wake=<gpio>[:0|1][|<gpio>[:0|1]...][,rtc=<gpio>[:0|1][|<gpio>[:0|1]...][,batt=<voltage>][,spurious=<mins>]
```
- delay: inactivity in **minutes** before going to sleep
- spurious: when using IR, wake-up can be triggered by any activity on the allocated GPIO, hence other remotes may cause unwanted wake-up. This sets (in **minutes** - default is 1) an inactivity delay after which sleep resumes.
- sleep: GPIO that will put the system into sleep and it can be a level 0 or 1.
- wake: **list** of GPIOs that with cause it to wake up (reboot) with their respective values. In such list, GPIO's are separated by an actual '|'.
- batt: threshold in **volts** under which the system will enter into sleep.

The battery voltage is measured every 10 seconds and 30 values are averaged before producing a result. The result must be 3 times below the threshold to enter sleep, so it takes a total of 10\*30\*3 = 15 minutes.

Be mindful that if the same GPIO is used to go to sleep and wakeup with the *same* level (in other word it's a transition/edge that triggers the action) the above will not work and the esp32 will immediately restart. In such case, you case use a button definition. The benefit of buttons is that not only can you re-use one actual button (e.g. 'stop') to make it the sleep trigger (using a long-press or a shift-press) but by selecting the ACTRLS_SLEEP action upon 'release', you can got to sleep upon release (1-0-1 transition) but also wake up upon another press (0 level applied on GPIO) because you only go to sleep *after* the GPIO returned to 1.

Please see [buttons](#buttons) for detailed syntax.

The option to use multiple GPIOs is very limited on esp32 and the esp-idf 4.3.x we are using: it is only possible to wake-up when **any** of the defined GPIO is set to 1. The fact that you can specify different levels in the wake list is irrelevant for now, it's just a provision for future upgrades to more recent versions of esp-idf.

**Only the following GPIOs can be used to wake-up the esp32**
- ESP32: 0, 2, 4, 12-15, 25-27, 32-39;
- ESP32-S3: 0-21.

Some have asked for a soft power on/off option. Although this is not built-in, it's easy to create yours as long as the regulator/power supply of the board can be controlled by Vcc or GND. Depending on how it is active, add a pull-up/down resistor to the regulator's control and connect it also to one GPIO of the esp32. Then using set_GPIO, set that GPIO to Vcc or GND. Use a hardware button that forces the regulator on with a pull- up/down and once the esp32 has booted, it will force the GPIO to the desired value maintaining the board on by software. To power it off by software, just use the deep sleep option which will suspend all GPIO hence switching off the regulator.

# Software configuration

## Setup WiFi
- Boot the esp, look for a new wifi access point showing up and connect to it. Default build ssid and passwords are "squeezelite"/"squeezelite". 
- Once connected, navigate to 192.168.4.1 
- Wait for the list of access points visible from the device to populate in the web page.
- Choose an access point and enter any credential as needed
- Once connection is established, note down the address the device received; this is the address you will use to configure it going forward 

## Setup squeezelite command line (optional)
At this point, the device should have disabled its built-in access point and should be connected to a known WiFi network.
- navigate to the address that was noted in step #1
- Using the list of predefined options, choose the mode in which you want squeezelite to start
- Generate the command
- Add or change any additional command line option (for example player name, etc)
- Activate squeezelite execution: this tells the device to automatiaclly run the command at start
- Update the configuration
- click on the "start toggle" button. This will force a reboot. 
- The toggle switch should be set to 'ON' to ensure that squeezelite is active after booting (you might have to fiddle with it a few times)
- You can enable accessto  NVS parameters under 'credits'

## Spotify
By default, SqueezeESP32 will use ZeroConf to advertise its Spotify capabilties. This means that until at least one local Spotify Connect application controllers discovers and connects to it, SqueezeESP32 will not be registered to Spotify servers. As a consequence, Spotify's WebAPI will not be able to see it (for example, Home Assistant services will miss it). Once you are connected to it using for example Spotify Desktop app, it will be registered and displayed everywhere.

If you want the player to be registered at start-up, you need to disable the ZeroConf option using the WebUI or `cspot_config::ZeroConf`. In that mode, the first time you run SqueezeESP32, it will be in ZeroConf mode and when you connect to it using a controller for the firt time, it receives and store credentials that will be used next time (after reboot). 

Set ZeroConf to 1 will always force ZeroConf mode to be used. 

The ZeroConf mode consumes less memory as it uses the built-in HTTP and mDNS servers to broadcast its capabilities. A Spotify controller will then discover these and trigger the SqueezeESP32 Spotify stack (cspot) to start. When the controller disconnects, the stack is shut down. In non-ZeroConf mode, the stack starts immediately (providing stored credentials are valid) and always run - a disconnect will not shut it down.

## Monitor
In addition of the esp-idf serial link monitor option, you can also enable a telnet server (see NVS parameters) where you'll have access to a ton of logs of what's happening inside the WROVER.

## Update Squeezelite
- From the firmware tab, click on "Check for Updates"
- Look for updated binaries
- Select a line
- Click on "Flash!"
- The system will reboot into recovery mode (if not already in that mode), wipe the squeezelite partition and download/flash the selected version 
- You can choose a local file or have a local webserver

## Recovery
- From the firmware tab, click on the "Recovery" button. This will reboot the ESP32 into recovery, where additional configuration options are available from the NVS editor

## Additional configuration notes (from the Web UI)
The squeezelite options are very similar to the regular Linux ones. Differences are :

	- the output is -o ["BT -n '<sinkname>' "] | [I2S]
	- if you've compiled with RESAMPLE option, normal soxr options are available using -R [-u <options>]. Note that anything above LQ or MQ will overload the CPU
	- if you've used RESAMPLE16, <options> are (b|l|m)[:i], with b = basic linear interpolation, l = 13 taps, m = 21 taps, i = interpolate filter coefficients

For example, so use a BT speaker named MySpeaker, accept audio up to 192kHz and resample everything to 44100 and use 16 bits resample with medium quality, the command line is:
	
	squeezelite -o "BT -n 'BT <sinkname>'" -b 500:2000 -R -u m -Z 192000 -r "44100-44100"

See squeezelite command line, but keys options are

	- Z <rate> : tell LMS what is the max sample rate supported before LMS resamples
	- R (see above)
	- r "<minrate>-<maxrate>"
	- C <sec> : set timeout to switch off amp gpio
	- W : activate WAV and AIFF header parsing
 	- s <name>|-disable: connect to a specific server. Use -disable to not search for any server

 **There is a safety feature to protect against WiFi/LMS connection loss that forces a reboot every few minutes when there is no LMS server detected. In case you don't want to use LMS at all, please set the server name to "-disable" on squeezelite command line ("-s -disable")**

# Building everything yourself

## Setting up ESP-IDF

### Docker
A simple alternative to building the project's binaries is to leverage the same docker image that is being used on the GitHub Actions to build our releases. The instructions below assume that you have cloned  the squeezelite-esp32 code that you want to build locally and that you have opened a command line/bash session in the folder that contains the code. 
Pull the most recent docker image for the environment: 
```
docker pull sle118/squeezelite-esp32-idfv435
```
Then run the container interactively :
```
for windows:
docker run -v %cd%:/project -w /project -it sle118/squeezelite-esp32-idfv435
for linux:
docker run -it -v `pwd`:/workspace/squeezelite-esp32 sle118/squeezelite-esp32-idfv435
```
The above command will mount this repo into the docker container and start a bash terminal. From there, simply run idf.py build to build, etc. Note that at the time of writing these lines, flashing is not possible for docker running under windows https://github.com/docker/for-win/issues/1018.

### Manual Install of ESP-IDF
You can install IDF manually on Linux or Windows (using the Subsystem for Linux) following the instructions at: https://www.instructables.com/id/ESP32-Development-on-Windows-Subsystem-for-Linux/ or see here https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/windows-setup.html for a direct install. You also need a few extra Python libraries for cspot by adding `[sudo] pip3 install protobuf grpcio-tools`

**Use the esp-idf 4.3.5 https://github.com/espressif/esp-idf/tree/release/v4.3.5 ** or the 4.4.5 (and above version) if you want to build for esp32-s3. You should download the a zip of the whole github tree (at the version you need) and runthe installer (`install.bat [esp32[,esp32s3]]` from there. Some Windows version (at least) have now a SSL certificate issue. You can workaround this by editing idf-tools.py and adding the following under ìmport ssl`
```
import ssl
ssl._create_default_https_context = ssl._create_unverified_context
```
And because the fun never ends, some Windows installations might fail to build a few files and spit a tons of errors on the output. It seems that the cache of the compile is a problem, so try to disable it by running `idf.py --no-ccache build` (I know...)
## Building SqueezeESP32
When initially cloning the repo, make sure you do it recursively. For example: `git clone --recursive https://github.com/sle118/squeezelite-esp32.git`. You also should install cspot additional components for protobuf use.
```
$ sudo pip3 install protobuf grpcio-tools
```
NB: I need to check on a fresh installation, but you might also require "protoc". You should do that within the esp32 local Python environment.
	
Don't forget to choose one of the config files in build_scripts/ and rename it sdkconfig.defaults or sdkconfig as many important WiFi/BT options are set there. **The codecs libraries will not be rebuilt by these scripts (it's a tedious process - see below)**

Create and tweak your config using `idf.py menuconfig` then build binaries using `idf.py all`. It will build the recovery and the application (squeezelite). then use `idf.py flash` to write everything. Otherwise, if you just want to download squeezelite, do (assuming you have set ESPPORT (e.g. COM10) and ESPBAUD (e.g. 921600)
```
<path_to_your_python>/python.exe <path_to_your_esptool>/esptool.py -p %ESPPORT% -b %ESPBAUD% --before default_reset --after hard_reset write_flash --flash_mode dio --flash_size detect --flash_freq 80m 0x150000 build/squeezelite.bin
```
Use `idf.py monitor` to monitor the application (see esp-idf documentation)

Note: You can use `idf.py build -DDEPTH=32` to build the 32 bits version and add the `-DVERSION=<your_version>` to add a custom version name (it will be 0.0-<your_version>). If you want to change the whole version string, see squeezelite.h. You can also disable the SBR extension of AAC codecs as it consumes a lot of CPU and might overload the esp32. Use `-DAAC_DISABLE_SBR=1` for that

If you have already cloned the repository and you are getting compile errors on one of the submodules (e.g. telnet), run the following git command in the root of the repository location: `git submodule update --init --recursive`

### Rebuild codecs (highly recommended to NOT try that)
- for codecs libraries, add -mlongcalls if you want to rebuild them, but you should not (use the provided ones in codecs/lib). if you really want to rebuild them, open an issue
- libmad, libflac (no esp's version), libvorbis (tremor - not esp's version), alac work
- libfaad does not really support real time, but if you want to try (but using helixaac is a better option)
	- -O3 -DFIXED_POINT -DSMALL_STACK
	- change ac_link in configure and case ac_files, remove ''
	- compiler but in cfft.c and cffti1, must disable optimization using
			#pragma GCC push_options
			#pragma GCC optimize ("O0")
			#pragma GCC pop_options
- opus & opusfile
	- for opus, the ESP-provided library seems to work, but opusfile is still needed
	- per mad & few others, edit configure and change $ac_link to add -c (faking link)
	- change ac_files to remove ''
	- add DEPS_CFLAGS and DEPS_LIBS to avoid pkg-config to be required
	- stack consumption can be very high with some codec variants, so set NONTHREADSAFE_PSEUDOSTACK and GLOBAL_STACK_SIZE=48000 and unset VAR_ARRAYS in config.h
- libmad has been patched to avoid using a lot of stack and is not provided here. There is an issue with sync detection in 1.15.1b from where the original stack patch was done but since a few fixes have been made wrt sync detection. This 1.15.1b-10 found on debian fixes the issue where mad thinks it has reached sync but has not and so returns a wrong sample rate. It comes at the expense of 8KB (!) of code where a simple check in squeezelite/mad.c that next_frame[0] is 0xff and next_frame[1] & 0xf0 is 0xf0 does the trick ...

# Hardware tips
There is a possibility to have a software on/off where a temporary switch can power-up the esp32 which then will auto-sustain its power. Depending on the selected hardware, it a can also include a power-off by using a long press on the same button. 

The auto-power is simply acheived by using `setGPIO` and forcing a GPIO to Vcc or GND and the sustain on/off requires a button creation whose longpress is an ACTRLS_SLEEP action (see also the [Sleeping](#sleeping) section). Credits [Renber78](http://github.com/Renber78) for schedmatics below

![alt text](https://github.com/sle118/squeezelite-esp32/blob/7eb4b218e31aa4692c5280fbec4619f690032c4a/Soft%20Power.png)

# Footnotes
