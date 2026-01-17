# Project 4

Display text from the Raspberry Pi Pico onto your character LCD.

Return to main menu [here](README.md)

## Toolchain

mpremote (no Thonny)  

## 🧰 Hardware Required

- Raspberry Pi Pico
- Your LCD + I²C backpack
- Breadboard
- Jumper wires

No resistors required for logic (already handled on backpack).

## Hard Reset for New Project

1. BOOTSEL to throw flash_nuke.uf2 onto pico
2. Unplug power to pico
3. BOOTSEL to throw [RPI_PICO-20251209-v1.27.0.uf2 ](https://micropython.org/resources/firmware/RPI_PICO-20251209-v1.27.0.uf2) onto pico
4. Unplug power to pico
5. Power to pico without BOOTSEL

### Whether to Continue

If the `ls -l` command below gives a result like example result below, you are safe to continue.

    lsusb
    ls -l /dev/ttyACM*

### Example Result

    [wgibbs@beta ~]$ ls -l /dev/ttyACM*
    crw-rw----. 1 root dialout 166, 0 Jan 17 13:40 /dev/ttyACM0
    
## 🔌 Step 1: Wire the LCD to the Pico (I²C)

### LCD Backpack Pins → Pico Pins
    
    LCD Pin	        Pico Pin	      Pico GPIO
    GND	            GND	            GND
    VCC	            3V3	            3.3V
    SDA	            GP0	            GPIO 0
    SCL	            GP1	            GPIO 1

⚠️ Do not use 5V.  Your backpack is clearly designed for 3.3V logic.

Wiring Checklist
- GND → GND
- VCC → 3V3
- SDA → GP0
- SCL → GP1

Do not proceed until wiring is solid.

<img src="img/PROJECT4.jpeg" alt="Photo" width="300" height="200">

## 🔍 Step 2: Verify Pico Can See the LCD (I²C Scan)

We always detect hardware before controlling it.

Connect to the Pico
    mpremote connect /dev/ttyACM0 repl

Run this I²C scan
    from machine import Pin, I2C

    i2c = I2C(0, scl=Pin(1), sda=Pin(0))
    print(i2c.scan())

Expected Result

You should see one address, commonly:

    39 (0x27) or
    
    63 (0x3F)

Example:

    [39]

❗ If you see [], stop — wiring or power is wrong.

## 🧩 Step 3: Load a Minimal LCD Driver (Manually)

MicroPython does not include an LCD driver by default.
We will copy one directly to the Pico.

Create a file: lcd_i2c.py

Paste this exact code (don’t modify yet):

    from machine import I2C
    import time
    
    class LCD:
      def __init__(self, i2c, addr, rows=2, cols=16):
          self.i2c = i2c
          self.addr = addr
          self.rows = rows
          self.cols = cols
          self.init_lcd()
  
      def write_byte(self, data):
          self.i2c.writeto(self.addr, bytes([data]))
  
      def toggle_enable(self, data):
          self.write_byte(data | 0b00000100)
          time.sleep_us(1)
          self.write_byte(data & ~0b00000100)
          time.sleep_us(50)
  
      def send(self, data, mode):
          high = mode | (data & 0xF0) | 0b00001000
          low = mode | ((data << 4) & 0xF0) | 0b00001000
          self.write_byte(high)
          self.toggle_enable(high)
          self.write_byte(low)
          self.toggle_enable(low)
  
      def command(self, cmd):
          self.send(cmd, 0)
  
      def write(self, char):
          self.send(ord(char), 1)
  
      def clear(self):
          self.command(0x01)
          time.sleep_ms(2)
  
      def init_lcd(self):
          time.sleep_ms(50)
          self.command(0x33)
          self.command(0x32)
          self.command(0x28)
          self.command(0x0C)
          self.command(0x06)
          self.clear()
  
      def print(self, text):
          for char in text:
              self.write(char)

Copy it to the Pico
        mpremote fs cp lcd_i2c.py :lcd_i2c.py

## 🧪 Step 4: First LCD Test (REPL)

Back in the REPL:

        from machine import Pin, I2C
        from lcd_i2c import LCD
        
        i2c = I2C(0, scl=Pin(1), sda=Pin(0))
        lcd = LCD(i2c, 0x27)   
        
        lcd.print("Hello Pico")

### 🎉 Expected Result

- LCD backlight ON
- Text appears:
    Hello Pico

If text is faint or invisible:
- Turn the blue potentiometer on the backpack slowly with a screwdriver
