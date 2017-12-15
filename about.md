---
layout: page
title: 정보
description: When building a website it's helpful to see what the focus of your site is. This page is an example of how to show a website's focus.
sitemap:
    priority: 0.7
    lastmod: 2017-11-02
    changefreq: weekly
---
## MVP에 대해

<span class="image left"><img src="{{ "/images/pic04.jpg" | absolute_url }}" alt="" /></span>

  오픈소스 SW수업 시간에 과제로 받은 MVP 만들기. 조그만한 가게에 놔둘 수 있는 간단한 음악 플레이어를 만드는 것이 주 목적이었다. 따라서 이 파트에서는 MVP가 어떠한 것들로 구성되어 있는지, 내가 참여한 부분이 무엇인지에 대해 자세히 설명하려 한다.
### 1. 코딩
<div class="box">
  <p>
  구현한 알고리즘은 간단하다.
  과제에서 주어진 (1)이전 곡, (2)다음 곡, (3)이전 채널, (4)다음 채널, (5)프로그램이 종료되었다가 다시 실행되었을 때 마지막에 재생했던 곡을 재생하도록 하는 기능, (6)부팅 시 프로그램 자동 실행(이는 라즈베리 파이의 local.rc파일을 수정하여 완성했다) 으로 나누어져 있다. 후에 셔플 재생 기능이 추가 되었지만 코드를 미리 백업을 해놓지 않은 탓에 이 코드에는 셔플 기능 구현이 되어있지 않다.

*참고 : write_____data 함수를 정의하기 전까지의 코드는 모두 LCD, 버튼, VLC(라즈베리파이의 음악 재생 프로그램) 등을 제어하기 위한 기본 세팅 코드다. 본인이 짠 코드는 아니라는 사실을 알린다.
  
  #!/usr/bin/env python
import vlc
import time
import RPi.GPIO as GPIO
import os
import smbus




GPIO.setmode(GPIO.BCM)
GPIO.setup(18, GPIO.IN)
GPIO.setup(23,GPIO.IN)
GPIO.setup(24,GPIO.IN)
GPIO.setup(25,GPIO.IN)

# Define some device parameters
I2C_ADDR = 0x27  # I2C device address
LCD_WIDTH = 16  # Maximum characters per line

# Define some device constants
LCD_CHR = 1  # Mode - Sending data
LCD_CMD = 0  # Mode - Sending command

LCD_LINE_1 = 0x80  # LCD RAM address for the 1st line
LCD_LINE_2 = 0xC0  # LCD RAM address for the 2nd line
LCD_LINE_3 = 0x94  # LCD RAM address for the 3rd line
LCD_LINE_4 = 0xD4  # LCD RAM address for the 4th line

LCD_BACKLIGHT = 0x08  # On
# LCD_BACKLIGHT = 0x00  # Off

ENABLE = 0b00000100  # Enable bit

# Timing constants
E_PULSE = 0.0005
E_DELAY = 0.0005

# Open I2C interface
# bus = smbus.SMBus(0)  # Rev 1 Pi uses 0
bus = smbus.SMBus(1)  # Rev 2 Pi uses 1


def lcd_init():
    # Initialise display
    lcd_byte(0x33, LCD_CMD)  # 110011 Initialise
    lcd_byte(0x32, LCD_CMD)  # 110010 Initialise
    lcd_byte(0x06, LCD_CMD)  # 000110 Cursor move direction
    lcd_byte(0x0C, LCD_CMD)  # 001100 Display On,Cursor Off, Blink Off
    lcd_byte(0x28, LCD_CMD)  # 101000 Data length, number of lines, font size
    lcd_byte(0x01, LCD_CMD)  # 000001 Clear display
    time.sleep(E_DELAY)


def lcd_byte(bits, mode):
    # Send byte to data pins
    # bits = the data
    # mode = 1 for data
    #        0 for command

    bits_high = mode | (bits & 0xF0) | LCD_BACKLIGHT
    bits_low = mode | ((bits << 4) & 0xF0) | LCD_BACKLIGHT

    # High bits
    bus.write_byte(I2C_ADDR, bits_high)
    lcd_toggle_enable(bits_high)

    # Low bits
    bus.write_byte(I2C_ADDR, bits_low)
    lcd_toggle_enable(bits_low)


def lcd_toggle_enable(bits):
    # Toggle enable
    time.sleep(E_DELAY)
    bus.write_byte(I2C_ADDR, (bits | ENABLE))
    time.sleep(E_PULSE)
    bus.write_byte(I2C_ADDR, (bits & ~ENABLE))
    time.sleep(E_DELAY)


def lcd_string(message, line):
    # Send string to display

    message = message.ljust(LCD_WIDTH, " ")

    lcd_byte(line, LCD_CMD)

    for i in range(LCD_WIDTH):
        lcd_byte(ord(message[i]), LCD_CHR)

------------------------------------------------------------------ 여기부터 본격적인 코드 시작

def write_data(x, y, z): #현재 재생중인 음악을 txt파일에 저장
    f = open("/home/pi/Desktop/temp.txt", 'w+')
    x = str(x)
    y = str(y)
    z = str(z)
    data = x+y+z
    f.write(data)
    f.close()

    
def play_music(): #음악 실행 루프를 돌리기 위한 가장 바깥쪽 함수
    if __name__ == '__main__':

        try:
            main()
        except KeyboardInterrupt:
            pass
        finally:
            lcd_byte(0x01, LCD_CMD)
        
        
def main():
    # Main program block

    # Initialise display
    lcd_init()

#음악이 저장된 music 폴더 속 폴더들을 스트링으로 반환
    dirname = "/home/pi/Desktop/music"
    dir = [f for f in os.listdir(dirname)
           if os.path.isdir(os.path.join(dirname, f))]

    path = []
    for name in dir: #각각의 폴더 경로들을 path리스트에 저장
        dirname = "/home/pi/Desktop/music"
        dirname += "/" + name
        path.append(dirname)
        
        #이중배열을 만들기 위해 path 리스트에 저장된 경로를 활용 (가장 안쪽 리스트에는 각 폴더 속 음악의
        #제목들이 들어가 있다.)
    a = []
    for i in range(len(path)):
        artist = os.listdir(path[i])
        a.append(artist)

#저장해 둔 txt파일을 열어 각 배열에 인덱스를 불러와 temp 변수에 경로를 지정해준다. 이로써 마지막으#로 재생했던 음악이 재생된다.
    f = open("/home/pi/Desktop/temp.txt")
    line = f.read()
    x = int(line[0])
    y = int(line[1])
    z = int(line[2])
    temp = path[x] + "/" + a[y][z]
    f.close()

    file = temp


    instance = vlc.Instance()

    player = instance.media_player_new()

    media = instance.media_new(file)

    player.set_media(media)

    player.play()

    while True: #본격적인 while 루프
        # Send some test
        split_path = path[x].split('/')
        lcd_artist = split_path[-1]
        lcd_title = a[y][z][:-4]
        lcd_string(">" + lcd_title, LCD_LINE_1)
        lcd_string(">" + lcd_artist, LCD_LINE_2)
        if GPIO.input(18) == 0: #previous song
            player.stop()
            z -= 1
            if (z == -1):
                z = len(a[y]) - 1
            time.sleep(0.6)
            write_data(x, y, z)
            play_music()
            time.sleep(0.4)
        elif GPIO.input(23) == 0: #previous channel
            player.stop()
            x -= 1
            y -= 1
            z = 0
            if (x == -1):
                x = len(path) - 1
            if (y == -1):
                y = len(a) - 1
            time.sleep(0.6)
            write_data(x, y, z)
            play_music()
            time.sleep(0.4)
        elif GPIO.input(24) == 0: #next song
            player.stop()
            z += 1
            if (z == len(a[y])):
                z = 0
            time.sleep(0.6)
            write_data(x, y, z)
            play_music()
            time.sleep(0.4)
        elif GPIO.input(25) == 0: #next channel
            player.stop()
            x += 1
            y += 1
            z = 0
            if (x == len(path)):
                x = 0
            if (y == len(a)):
                y = 0
            time.sleep(0.6)
            write_data(x, y, z)
            play_music()
            time.sleep(0.4)


play_music()

  </p>
</div>

### 2. 디자인 및 모델링
<span class="image left"><img src="{{ "/images/pic05.jpg" | absolute_url }}" alt="" /></span>

<div class="box">
  <p>
디자인 및 모델링 부분은 나의 관여가 거의 없었지만 3D프린터로 출력할 때의 사이즈 조정, 밀도, 두께 등 세부사항을 설정하여 직접 프린트하는 역할을 했다. 디자인이 화려하여 더욱 출력하기 어려웠고 사이즈가 생각보다 작아 의도치 않게 PCB를 사용하게 되어 불가피하게 도움을 받았지만 기존에 헷갈려했던 회로 연결에 대해 확실히 알게 되는 계기가 되었다.
  </p>
</div>

###3. 발표

<div class="box">
  <p>
발표 파트는 자세히 말로 설명하기는 부족하니 영상으로 대체하도록 하겠다.

[![Video Label]("/images/pic06.jpg")](http://101.79.235.31/redirect/video.nmv.naver.com/blog/blog_2017_12_15_111/b5efac7db04c73ce0791404de71307ba874d_ugcvideo_480P_01.mp4?key=MjEwMzIyMjA1MzQ2OTcwNDQzMTE0MzExMTAxOXZpZGVvLm5tdi5uYXZlci5jb20wODMvYmxvZy9ibG9nXzIwMTdfMTJfMTVfMTExL2I1ZWZhYzdkYjA0YzczY2UwNzkxNDA0ZGU3MTMwN2JhODc0ZF91Z2N2aWRlb180ODBQXzAxLm1wNDMxMjEyMTgwMDhzaW1vbmlkYTIyODMxODhOSE5NVjAwMDAwMDQzMjUwNDI3Mjk=&px-bps=1632448&px-bufahead=3&in_out_flag=0)
  </p>
</div>
