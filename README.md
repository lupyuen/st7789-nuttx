# ST7789 Demo App for Apache NuttX RTOS

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

# Configure NuttX

Configure NuttX with menuconfig...

Enable SPI CMD/DATA:
- Device Drivers → SPI Driver Support → SPI CMD/DATA

Enable ST7789 Driver:
- Device Drivers → LCD Driver Support → Graphic LCD Driver Support → LCD driver selection → Sitronix ST7789 TFT Controller 
 
Enable LCD Character Device:
- Device Drivers → LCD Driver Support → Graphic LCD Driver Support LCD → character device   

Enable LVGL Demo App:
- Application Configuration → Graphics Support → Light and Versatile Graphic Library (LVGL)
- Graphics Settings → Horizontal Resolution: Set to 240
- Graphics Settings → Vertical Resolution: Set to 240

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

TODO

Enable SPI Master and clear FIFO in Poll Send:

https://github.com/lupyuen/incubator-nuttx/pull/42

# SPI Cmd/Data

TODO

Implement SPI Cmd/Data:

https://github.com/lupyuen/incubator-nuttx/pull/44

After fixing SPI Send and SPI Cmd/Data...

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
nsh>
```

# Render Pink Screen

Let's render a pink screen on ST7789:

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

# LVGL

TODO
