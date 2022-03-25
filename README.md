# ST7789 Demo App for Apache NuttX RTOS

(Tested on Pine64 PineCone BL602)

TODO

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
