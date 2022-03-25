# ST7789 and LVGL Demo for Apache NuttX RTOS

(Tested on Pine64 PineCone BL602)

ST7789 Display and LVGL Graphics... Let's make it work on Apache NuttX RTOS!

# Connect BL602 to ST7789

Connect BL602 to ST7789 as follows...

| BL602 Pin     | ST7789 SPI          | Wire Colour 
|:--------------|:--------------------|:-------------------
| __`GPIO 0`__  | `DC`  _(MISO)_      | Blue
| __`GPIO 2`__  | Unused _(CS)_
| __`GPIO 1`__  | `SDA` _(MOSI)_      | Yellow
| __`GPIO 3`__  | `SCL` _(SCK)_       | Green
| __`GPIO 4`__  | `RST`               | Black
| __`GPIO 5`__  | `BLK`               | Orange
| __`3V3`__     | `3.3V`              | Grey
| __`GND`__     | `GND`               | Black

TODO: Photo

# Configure NuttX

Configure NuttX with menuconfig...

Enable SPI CMD/DATA:
- Device Drivers → SPI Driver Support → SPI CMD/DATA

Enable ST7789 Driver:
- Device Drivers → LCD Driver Support → Graphic LCD Driver Support → LCD driver selection → Sitronix ST7789 TFT Controller 
- X Resolution: Set to 240
- Y Resolution: Set to 240
- SPI Mode: See below
 
Enable LCD Character Device:
- Device Drivers → LCD Driver Support → Graphic LCD Driver Support LCD → character device   

Enable LVGL Library:
- Application Configuration → Graphics Support → Light and Versatile Graphic Library (LVGL)
- Graphics Settings → Horizontal Resolution: Set to 240
- Graphics Settings → Vertical Resolution: Set to 240

Enable LVGL Demo App:
- Application Configuration → Examples → LVGL Demo

Enable Logging:
- Build Setup → Debug Options 
- Select:
  - Graphics Debug Features
  - Graphics Error Output
  - Graphics Warnings Output
  - Graphics Informational Output  
  - Low-level LCD Debug Features
  - LCD Driver Error Output
  - LCD Driver Warnings Output
  - LCD Driver Informational Output

[(.config for BL602)](https://gist.github.com/lupyuen/a7fc921531c1d14e8d336fba0cdb1c83)

# SPI Mode 3

ST7789 only works with BL602 in SPI Mode 3 for some unknown reason...

```c
#ifdef CONFIG_BL602_SPI0
#  warning Using SPI Mode 3 for ST7789 on BL602
#  define CONFIG_LCD_ST7789_SPIMODE SPIDEV_MODE3 /* SPI Mode 3: Workaround for BL602 */
#else
#  ifndef CONFIG_LCD_ST7789_SPIMODE
#    define CONFIG_LCD_ST7789_SPIMODE SPIDEV_MODE0
#  endif /* CONFIG_LCD_ST7789_SPIMODE */
#endif  /* CONFIG_BL602_SPI0 */
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/st7789/drivers/lcd/st7789.c#L50-L57)

[(More about this)](https://lupyuen.github.io/articles/display#initialise-spi-port)

# Fix SPI Send

We enable SPI Master and clear FIFO in SPI Poll Send:

https://github.com/lupyuen/incubator-nuttx/pull/42

Logic Analyser shows that SPI Poll Send now transmits SPI Data correctly:

TODO: Screenshot

# SPI Cmd/Data

We implement SPI Cmd/Data for BL602:

https://github.com/lupyuen/incubator-nuttx/pull/44

After fixing SPI Send and SPI Cmd/Data, the ST7789 Driver should start properly without hanging...

```text
▒gpio_pin_register: Registering /dev/gpio0
gpio_pin_register: Registering /dev/gpio1
gpint_enable: Disable the interrupt
gpio_pin_register: Registering /dev/gpio2
bl602_gpio_set_intmod: ****gpio_pin=115, int_ctlmod=1, int_trgmod=0
bl602_spi_setfrequency: frequency=400000, actual=0
bl602_spi_setbits: nbits=8
bl602_spi_setmode: mode=0
st7789_sleep: sleep: 0
st7789_sendcmd: cmd: 0x11
bl602_spi_select: devid: 262144, CS: select
bl602_spi_setmode: mode=3
bl602_spi_setbits: nbits=8
bl602_spi_setfrequency: frequency=1000000, actual=0
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=1
bl602_spi_poll_send: send=11 and recv=0
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=0
bl602_spi_select: devid: 262144, CS: free
bl602_spi_select: Revert MISO to SPIst7789_sendcmd: OK
st7789_bpp: bpp: 16
st7789_sendcmd: cmd: 0x3a
bl602_spi_select: devid: 262144, CS: select
bl602_spi_setmode: mode=3
bl602_spi_setbits: nbits=8
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=1
bl602_spi_poll_send: send=3a and recv=0
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=0
bl602_spi_select: devid: 262144, CS: free
bl602_spi_select: Revert MISO to SPIst7789_sendcmd: OK
bl602_spi_select: devid: 262144, CS: select
bl602_spi_setmode: mode=3
bl602_spi_setbits: nbits=8
bl602_spi_poll_send: send=5 and recv=ff
bl602_spi_select: devid: 262144, CS: free
bl602_spi_select: Revert MISO to SPIst7789_setorientation:
st7789_sendcmd: cmd: 0x36
bl602_spi_select: devid: 262144, CS: select
bl602_spi_setmode: mode=3
bl602_spi_setbits: nbits=8
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=1
bl602_spi_poll_send: send=36 and recv=0
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=0
bl602_spi_select: devid: 262144, CS: free
bl602_spi_select: Revert MISO to SPIst7789_sendcmd: OK
bl602_spi_select: devid: 262144, CS: select
bl602_spi_setmode: mode=3
bl602_spi_setbits: nbits=8
bl602_spi_poll_send: send=70 and recv=ff
bl602_spi_select: devid: 262144, CS: free
bl602_spi_select: Revert MISO to SPIst7789_display: on: 1
st7789_sendcmd: cmd: 0x29
bl602_spi_select: devid: 262144, CS: select
bl602_spi_setmode: mode=3
bl602_spi_setbits: nbits=8
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=1
bl602_spi_poll_send: send=29 and recv=0
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=0
bl602_spi_select: devid: 262144, CS: free
bl602_spi_select: Revert MISO to SPIst7789_sendcmd: OK
st7789_sendcmd: cmd: 0x21
bl602_spi_select: devid: 262144, CS: select
bl602_spi_setmode: mode=3
bl602_spi_setbits: nbits=8
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=1
bl602_spi_poll_send: send=21 and recv=0
bl602_spi_cmddata: Change MISO to GPIO, devid=262144, cmd=0
bl602_spi_select: devid: 262144, CS: free
bl602_spi_select: Revert MISO to SPIst7789_sendcmd: OK
board_lcd_getdev: SPI port 0 bound to LCD 0
st7789_getplaneinfo: planeno: 0 bpp: 16

NuttShell (NSH) NuttX-10.2.0-RC0
nsh> ls /dev
/dev:
 console
 gpio0
 gpio1
 gpio2
 lcd0
 null
 spi0
 spitest0
 timer0
 urandom
 zero
```

Note that our ST7789 Display appears as "__/dev/lcd0__". This will be used by the LVGL Library.

# Render Pink Screen

Let's render a pink screen at startup on ST7789:

```c
FAR struct lcd_dev_s *st7789_lcdinitialize(FAR struct spi_dev_s *spi)
{
  ...
  st7789_sleep(priv, false);
  st7789_bpp(priv, ST7789_BPP);
  st7789_setorientation(priv);
  st7789_display(priv, true);
  
  #warning Render Pink Screem
  st7789_fill(priv, 0xaaaa);
  //  Previously: st7789_fill(priv, 0xffff);
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/st7789/drivers/lcd/st7789.c#L752-L777)

When NuttX boots, we should see a pink screen. The screen refresh looks laggy, hopefully we'll fix this with SPI DMA.

TODO: Screenshot

[(Watch the demo on YouTube)](https://youtu.be/iaDYjO1rCmY)

# Run LVGL Demo

To run the LVGL Demo App...

```text
▒gpio_pin_register: Registering /dev/gpio0
gpio_pin_register: Registering /dev/gpio1
gpint_enable: Disable the interrupt
gpio_pin_register: Registering /dev/gpio2
bl602_gpio_set_intmod: ****gpio_pin=115, int_ctlmod=1, int_trgmod=0
spi_test_driver_register: devpath=/dev/spitest0, spidev=0
st7789_sleep: sleep: 0
st7789_sendcmd: cmd: 0x11
st7789_sendcmd: OK
st7789_bpp: bpp: 16
st7789_sendcmd: cmd: 0x3a
st7789_sendcmd: OK
st7789_setorientation:
st7789_sendcmd: cmd: 0x36
st7789_sendcmd: OK
st7789_display: on: 1
st7789_sendcmd: cmd: 0x29
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x21
st7789_sendcmd: OK
st7789_fill: color: 0xaaaa
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
board_lcd_getdev: SPI port 0 bound to LCD 0
st7789_getplaneinfo: planeno: 0 bpp: 16

NuttShell (NSH) NuttX-10.2.0-RC0
nsh> ?
help usage:  help [-v] [<cmd>]

  ?      cat    help   ls     uname

Builtin Apps:
  tinycbor_test            spi_test                 nsh
  lorawan_test             timer                    sensortest
  sx1262_test              bl602_adc_test           ikea_air_quality_sensor
  bas                      spi_test2                gpio
  sh                       getprime
  lvgldemo                 hello
nsh> lvgldemo
fbdev_init: Failed to open /dev/fb0: 2
st7789_getvideoinfo: fmt: 11 xres: 240 yres: 240 nplanes: 1
lcddev_init: VideoInfo:
        fmt: 11
        xres: 240
        yres: 240
        nplanes: 1
lcddev_init: PlaneInfo (plane 0):
        bpp: 16
st7789_putarea: row_start: 0 row_end: 19 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 20 row_end: 39 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 40 row_end: 59 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 60 row_end: 79 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 80 row_end: 99 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 100 row_end: 119 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 120 row_end: 139 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 140 row_end: 159 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 160 row_end: 179 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 180 row_end: 199 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 200 row_end: 219 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 220 row_end: 239 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
monitor_cb: 57600 px refreshed in 1100 ms
```

The LVGL Demo Screen appears on ST7789 yay!

TODO: Screenshot

# LVGL Version

Note that NuttX builds with LVGL version 7.3.0. The current version of LVGL is 8.2.0.

LVGL 8 is NOT backward compatible with LVGL 7. See the changes...

https://github.com/lvgl/lvgl/blob/master/docs/CHANGELOG.md#v800-01062021

Should we migrate to LVGL 8? Or upgrade to LVGL 7.11.0?
