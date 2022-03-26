![Connect BL602 to ST7789](https://lupyuen.github.io/images/st7789-title.jpg)

# ST7789 and LVGL Demo for Apache NuttX RTOS

(Tested on Pine64 PineCone BL602)

ST7789 Display with LVGL Graphics... Let's make it work on Apache NuttX RTOS!

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

![Connect BL602 to ST7789](https://lupyuen.github.io/images/st7789-connect.jpg)

The BL602 pins for ST7789 are defined in [board.h](https://github.com/lupyuen/incubator-nuttx/blob/st7789/boards/risc-v/bl602/bl602evb/include/board.h#L92-L104)

```c
/* SPI Configuration: For PineCone BL602 */

#define BOARD_SPI_CS   (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN2)
#define BOARD_SPI_MOSI (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN1)
#define BOARD_SPI_MISO (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN0)
#define BOARD_SPI_CLK  (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN3)

#ifdef CONFIG_LCD_ST7789
/* ST7789 Configuration: Reset and Backlight Pins */

#define BOARD_LCD_RST (GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO | GPIO_PIN4)
#define BOARD_LCD_BL  (GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO | GPIO_PIN5)
#endif  /* CONFIG_LCD_ST7789 */
```

# Configure NuttX

Configure NuttX with menuconfig...

Enable SPI Port:
- System Type → BL602 Peripheral Support → SPI0

Enable SPI CMD/DATA:
- Device Drivers → SPI Driver Support
  - SPI Exchange
  - SPI CMD/DATA
  - SPI Character Driver

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

![Fix SPI Send](https://lupyuen.github.io/images/st7789-spi2a.png)

# Fix SPI Send

On BL602, SPI Poll Send `bl602_spi_poll_send()` doesn't send any data because it doesn't enable SPI Master and it doesn't clear the SPI FIFO.

SPI Poll Send also hangs because it loops forever waiting for the SPI FIFO: [bl602_spi.c](https://github.com/apache/incubator-nuttx/blob/master/arch/risc-v/src/bl602/bl602_spi.c#L779-L803)

```c
static uint32_t bl602_spi_poll_send(struct bl602_spi_priv_s *priv, uint32_t wd)
{
  uint32_t val;
  uint32_t tmp_val = 0;

  /* write data to tx fifo */

  putreg32(wd, BL602_SPI_FIFO_WDATA);
  
  /* This loop hangs because SPI Master is not enabled and SPI FIFO is not cleared */
  
  while (0 == tmp_val)
    {
      /* get data from rx fifo */

      tmp_val = getreg32(BL602_SPI_FIFO_CFG_1);
      tmp_val = (tmp_val & SPI_FIFO_CFG_1_RX_CNT_MASK)
                >> SPI_FIFO_CFG_1_RX_CNT_SHIFT;
    }
```

This problem affects the NuttX ST7789 Driver because the ST7789 Driver calls SPI Poll Send via `SPI_SEND()` and `bl602_spi_send()`.

We fix this problem by moving the code that enables SPI Master and clears the FIFO, from SPI Poll Exchange `bl602_spi_poll_exchange()` to SPI Poll Send. (Note that SPI Poll Exchange calls SPI Poll Send)

Before fixing, SPI Poll Send looks like this: [bl602_spi.c](https://github.com/apache/incubator-nuttx/blob/master/arch/risc-v/src/bl602/bl602_spi.c#L779-L803)

```c
static uint32_t bl602_spi_poll_send(struct bl602_spi_priv_s *priv, uint32_t wd)
{
  uint32_t val;
  uint32_t tmp_val = 0;

  /* write data to tx fifo */

  putreg32(wd, BL602_SPI_FIFO_WDATA);

  while (0 == tmp_val)
    {
      /* get data from rx fifo */
      ...
```

After fixing, SPI Poll Send enables SPI Master and clears SPI FIFO: [bl602_spi.c](https://github.com/lupyuen/incubator-nuttx/blob/st7789/arch/risc-v/src/bl602/bl602_spi.c#L806-L839)

```c
static uint32_t bl602_spi_poll_send(struct bl602_spi_priv_s *priv, uint32_t wd)
{
  uint32_t val;
  uint32_t tmp_val = 0;

  /* spi enable master */

  modifyreg32(BL602_SPI_CFG, SPI_CFG_CR_S_EN, SPI_CFG_CR_M_EN);

  /* spi fifo clear  */

  modifyreg32(BL602_SPI_FIFO_CFG_0, SPI_FIFO_CFG_0_RX_CLR
              | SPI_FIFO_CFG_0_TX_CLR, 0);

  /* write data to tx fifo */

  putreg32(wd, BL602_SPI_FIFO_WDATA);

  while (0 == tmp_val)
    {
      /* get data from rx fifo */
      ...
```

Logic Analyser shows that SPI Poll Send now transmits SPI Data correctly:

![SPI Poll Send transmits SPI Data correctly](https://lupyuen.github.io/images/st7789-logic3.png)

Note that the MOSI Pin shows the correct data. Before fixing, the data was missing.

As for the modified SPI Poll Exchange, we tested it with Semtech SX1262 SPI Transceiver on PineCone BL602: [release-2022-03-25](https://github.com/lupyuen/incubator-nuttx/releases/tag/release-2022-03-25)

[(More about this)](https://github.com/lupyuen/incubator-nuttx/pull/42)

# SPI Cmd/Data

To control ST7789 Data / Command Pin, we implement SPI Cmd/Data for BL602:

https://github.com/lupyuen/incubator-nuttx/pull/44

After implementing SPI Cmd/Data, Logic Analyser shows that MISO goes Low when transmitting ST7789 Commands...

![MISO goes Low when transmitting ST7789 Commands](https://lupyuen.github.io/images/st7789-logic1.png)

And MISO goes High when transmitting ST7789 Data...

![MISO goes High when transmitting ST7789 Data](https://lupyuen.github.io/images/st7789-logic2a.png)

# SPI Mode 3

ST7789 only works with BL602 in SPI Mode 3 for some unknown reason: [st7789.c](https://github.com/lupyuen/incubator-nuttx/blob/st7789/drivers/lcd/st7789.c#L50-L57)

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

[(More about this)](https://lupyuen.github.io/articles/display#initialise-spi-port)

# Load ST7789 Driver

We load the LCD Driver at startup: [bl602_bringup.c](https://github.com/lupyuen/incubator-nuttx/blob/st7789/boards/risc-v/bl602/bl602evb/src/bl602_bringup.c#L676-L696)

```c
#ifdef CONFIG_LCD_DEV
#  include <nuttx/board.h>
#  include <nuttx/lcd/lcd_dev.h>
#endif

#ifdef CONFIG_LCD_ST7789
#include <nuttx/lcd/st7789.h>
#include "../boards/risc-v/bl602/bl602evb/include/board.h"
#include "riscv_internal.h"
#endif /* CONFIG_LCD_ST7789 */
...
int bl602_bringup(void) {
  ...
#ifdef CONFIG_LCD_DEV

  /* Initialize the LCD driver */

  ret = board_lcd_initialize();
  if (ret < 0)
    {
      _err("ERROR: board_lcd_initialize() failed: %d\n", ret);
    }

  /* Register the LCD driver */

  ret = lcddev_register(0);
  if (ret < 0)
    {
      _err("ERROR: lcddev_register() failed: %d\n", ret);
    }
#endif /* CONFIG_LCD_DEV */

  return ret;
}
```

And we load the ST7789 Driver at startup: [bl602_bringup.c](https://github.com/lupyuen/incubator-nuttx/blob/st7789/boards/risc-v/bl602/bl602evb/src/bl602_bringup.c#L698-L769)

```c
#ifdef CONFIG_LCD_ST7789

/* SPI Port Number for LCD */
#define LCD_SPI_PORTNO 0

/* SPI Bus for LCD */
static struct spi_dev_s *st7789_spi_bus;

/* LCD Device */
static struct lcd_dev_s *g_lcd = NULL;

/****************************************************************************
 * Name:  board_lcd_initialize
 *
 * Description:
 *   Initialize the LCD video hardware.  The initial state of the LCD is
 *   fully initialized, display memory cleared, and the LCD ready to use, but
 *   with the power setting at 0 (full off).
 *
 ****************************************************************************/

int board_lcd_initialize(void)
{
  st7789_spi_bus = bl602_spibus_initialize(LCD_SPI_PORTNO);
  if (!st7789_spi_bus)
    {
      lcderr("ERROR: Failed to initialize SPI port %d for LCD\n", LCD_SPI_PORTNO);
      return -ENODEV;
    }

  /* Pull LCD_RESET high */

  bl602_configgpio(BOARD_LCD_RST);
  bl602_gpiowrite(BOARD_LCD_RST, false);
  up_mdelay(1);
  bl602_gpiowrite(BOARD_LCD_RST, true);
  up_mdelay(10);

  /* Set full brightness */

  bl602_configgpio(BOARD_LCD_BL);
  bl602_gpiowrite(BOARD_LCD_BL, true);

  return OK;
}

/****************************************************************************
 * Name:  board_lcd_getdev
 *
 * Description:
 *   Return a a reference to the LCD object for the specified LCD.  This
 *   allows support for multiple LCD devices.
 *
 ****************************************************************************/

FAR struct lcd_dev_s *board_lcd_getdev(int devno)
{
  g_lcd = st7789_lcdinitialize(st7789_spi_bus);
  if (!g_lcd)
    {
      lcderr("ERROR: Failed to bind SPI port %d to LCD %d\n", LCD_SPI_PORTNO,
      devno);
    }
  else
    {
      lcdinfo("SPI port %d bound to LCD %d\n", LCD_SPI_PORTNO, devno);
      return g_lcd;
    }

  return NULL;
}
#endif  //  CONFIG_LCD_ST7789
```

# Boot NuttX

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

Let's render a pink screen at startup: [st7789.c](https://github.com/lupyuen/incubator-nuttx/blob/st7789/drivers/lcd/st7789.c#L752-L777)

```c
FAR struct lcd_dev_s *st7789_lcdinitialize(FAR struct spi_dev_s *spi) {
  ...
  st7789_sleep(priv, false);
  st7789_bpp(priv, ST7789_BPP);
  st7789_setorientation(priv);
  st7789_display(priv, true);
  
  #warning Render Pink Screen
  st7789_fill(priv, 0xaaaa);
  //  Previously: st7789_fill(priv, 0xffff);
```

When NuttX boots, we should see a pink screen. The screen refresh looks laggy, hopefully we'll fix this with SPI DMA.

[(Watch the demo on YouTube)](https://youtu.be/iaDYjO1rCmY)

![When NuttX boots, we should see a pink screen](https://lupyuen.github.io/images/st7789-boot.jpg)

# Run LVGL Demo

We run the LVGL Demo App `lvgldemo` with ST7789...

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

The LVGL Demo Screen appears on ST7789 yay! The demo source code is here: [lvgldemo.c](https://github.com/lupyuen/incubator-nuttx-apps/blob/st7789/examples/lvgldemo/lvgldemo.c)

![LVGL Demo Screen appears on ST7789](https://lupyuen.github.io/images/st7789-lvgl.jpg)

# LVGL Version

We can specify the LVGL Version in menuconfig:

Application Configuration → Graphics Support → Light and Versatile Graphic Library (LVGL) → LVGL Version  

Note that NuttX builds with LVGL version 7.3.0. The current version of LVGL is 8.2.0.

LVGL 8 is NOT backward compatible with LVGL 7. See the breaking changes...

- [LVGL 8.0.0 Release Notes](https://github.com/lvgl/lvgl/blob/master/docs/CHANGELOG.md#v800-01062021)

Should we migrate to LVGL 8? Or upgrade to LVGL 7.11.0?

_What happens if we switch to LVGL 7.11.0?_

The LVGL Demo App renders correctly when we switch to LVGL 7.11.0. But we're not sure if all the features are OK until we do a thorough test.

_What happens if we switch to LVGL 8.2.0?_

NuttX Build fails when downloading the LVGL Source Code...

```text
make[3]: Entering directory '/home/user/nuttx/apps/examples/lvgldemo'
Downloading: v8.2.0.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   118  100   118    0     0    361      0 --:--:-- --:--:-- --:--:--   360
100    14  100    14    0     0     12      0  0:00:01  0:00:01 --:--:--     0
Unpacking: v8.2.0.zip -> lv_demos
Archive:  v8.2.0.zip
  End-of-central-directory signature not found.  Either this file is not
  a zipfile, or it constitutes one disk of a multi-part archive.  In the
  latter case the central directory and zipfile comment will be found on
  the last disk(s) of this archive.
unzip:  cannot find zipfile directory in one of v8.2.0.zip or
        v8.2.0.zip.zip, and cannot find v8.2.0.zip.ZIP, period.
```

Seems the NuttX Makefiles need to be fixed to support LVGL 8. 

Also the demo source code needs to be updated for LVGL 8, since it's not compatible with LVGL 7: [lvgldemo.c](https://github.com/lupyuen/incubator-nuttx-apps/blob/st7789/examples/lvgldemo/lvgldemo.c)

# Sharing SPI Bus

PineDio Stack BL604 connects multiple SPI Devices to the same SPI Bus: ST7789 Display, SX1262 LoRa Transceiver, SPI Flash.

How to share the same SPI Bus for multiple SPI Devices? How do we control the Chip Select Pins for each SPI Device?

TODO
