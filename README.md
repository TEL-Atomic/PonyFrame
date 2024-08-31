# aioPonyFrame
TinyFrame is a simple library for building and parsing data frames to be sent over a serial interface (e.g. UART, telnet, socket). The code is written to build with --std=gnu99 and mostly compatible with --std=gnu89.

[PonyFrame](https://github.com/MightyPork/PonyFrame) is a port of TinyFrame to Python

aioPonyFrame is a fork of PonyFrame for use with asyncio

Overall there might be missing pieces and holes, but it gets at least some jobs done.

The library provides a high level interface for passing messages between the two peers. Multi-message sessions, response listeners, checksums, timeouts are all handled by the library.

TinyFrame is suitable for a wide range of applications, including inter-microcontroller communication, as a protocol for FTDI-based PC applications or for messaging through UDP packets.

The library lets you register listeners (callback functions) to wait for (1) any frame, (2) a particular frame Type, or (3) a specific message ID. This high-level API is general enough to implement most communication patterns.

Usage: (tested on circuitpython 9.1.1 on the Xiao RP2040)

first enable serial communications, not just the console. Add the following to boot.py
```
import usb_cdc
usb_cdc.enable(console=True, data=True)
```

```python
code.py:


import asyncio
import usb_cdc
import aioTinyFrame
import json
import math

class aioPonyFrame_example():
    def __init__(self):
        self.ser = usb_cdc.data
        self.ser.timeout=0

        self.TYPE_RESPONSE = 0x0000
        self.TYPE_STATUS = 0x0001
        self.TYPE_INSTRUCTION = 0x0002
        self.TYPE_ERROR = 0x0003

        self.tf = aioTinyFrame.TinyFrame()
        self.tf.TYPE_BYTES = 0x02
        self.tf.CKSUM_TYPE = 'crc16'
        self.tf.SOF_BYTE = 0x55
        # if you use some other serial library you will need to replace self.ser.write
        self.tf.write = self.ser.write 
        self.tf.add_fallback_listener(fallback_listener)
        self.tf.add_type_listener(self.TYPE_INSTRUCTION, self.instruction_listener)

        self.error_status = False
        self.data_type='sin'
        
    async def input_comms(self):
        while True:
            await asyncio.sleep(.1)
            #the emulated serial port on the RP2040 is very fast compared to the messages I am sending
            #this just reads the entire buffer. there is nothing to look for newlines or end of messages
            data_in = self.ser.read()
            if data_in != b'':
                await self.tf.accept(data_in)
                
    async def send_fake_data(self):
      n=0
      while True:
        await asyncio.sleep(1.5)
        if not self.error_status:
            if self.data_type == 'sin':
                value = {'sine data' : math.sin(n/100)} #read a sensor or something here
            elif self.data_type == 'cos':
                value = {'cosine data' : math.cos(n/100)} #read a different sensor...
                                    #something had to change for the example
            else:
                value = {'error':'invalid data type requested'}
                self.tf.query(self.TYPE_ERROR, self.error_id_listener, json.dumps(value).encode('UTF-8'))
                # a query expects a response (which should be sent automatically by tinyframe
                self.error_status = True
                continue
            self.tf.send(self.TYPE_STATUS, json.dumps(value).encode('UTF-8'))
        else:
            self.tf.send(self.TYPE_STATUS, json.dumps({'status':'error'}).encode('UTF-8'))
        n+=1
        
    async def main(self):
        input_comms_task = asyncio.create_task(self.input_comms())
        fake_data_task = asyncio.create_task(self.send_fake_data())
        await asyncio.gather(input_comms_task, fake_data_task)

    async def instruction_listener(self, TF, frame):
        # tinyframe will pass itself to this listener, so we need three positional arguments
        # let the computer know we received the message by returning a frame with the same ID
        self.tf.send(self.TYPE_INSTRUCTION,id=frame.id )
    
        dict = json.loads(frame.data)
        if 'data_type' in dict:
            self.data_type = dict['data_type']
            print(f'data type chaned to {self.data_type}')
            self.tf.send(self.TYPE_STATUS,json.dumps({'data_type':self.data_type}).encode('UTF-8'))

    async def error_id_listener(self, tf, frame):
      print('error acknowledged, resetting to default behavior') 
      # this is listening for a response from the computer acknowledging the error
      self.data_type = 'sin'
      self.error_status = False
      
# listeners can be in the class or outside, just adjust the number of arguments accordingly

async def fallback_listener(self, frame):
  # self in this case refers to the object tf, not to the aioPonyFrame_example object
  print('unexpected frame received, fallback listener activated')
  print(f'frame: {frame}')

example = aioPonyFrame_example()
asyncio.run(example.main())

```   

Below is a basic computer program to interface with the above microcontroller program. 
It uses PonyFrame, not aioPonyFrame and runs on windows.
it could be used with threading or multiprocessing (and a pipe or queue) to 
provide communication functions to some larger program

```python
import TinyFrame
import serial
import serial.tools.list_ports
import time
import json
import keyboard

TYPE_RESPONSE = 0x0000
TYPE_STATUS = 0x0001
TYPE_INSTRUCTION = 0x0002
TYPE_ERROR = 0x0003 

class PonyFrame_example():
    def __init__(self):
        print('press space to send command')
        self.stop = False

    def send_receive(self):
        while True:
            ports = serial.tools.list_ports.comports()
            for current_port in ports:
                try:
                    with serial.Serial(port=current_port[0],
                        baudrate=19600,
                        parity=serial.PARITY_NONE,
                        stopbits=serial.STOPBITS_ONE,
                        bytesize=serial.EIGHTBITS) as ser:                        
                        ser.write_timeout=0
                        ser.timeout = 0

                        print(f'connected on {current_port}')

                        self.tf=TinyFrame.TinyFrame()
                        self.tf.TYPE_BYTES = 0x02
                        self.tf.CKSUM_TYPE = 'crc16'
                        self.tf.SOF_BYTE = 0x55
                        self.tf.write = ser.write
                        self.tf.add_type_listener(TYPE_ERROR, self.error_listener)
                        self.tf.add_type_listener(TYPE_STATUS, self.status_listener)

                        timeout = time.time(
                        message = ''

                        while True:
                            # this is a horrible way to get input, but it works for this demonstration
                            # the buffer is quite large and will store lots of incoming data while we
                            # wait for input
                            if keyboard.is_pressed('space'):
                                message = input('What Type of data would you like to see?').strip()
                                self.tf.query(TYPE_INSTRUCTION, self.id_listener, json.dumps({'data_type':message}).encode('UTF-8'))
                                timeout = time.time()
                            # ser.read can be any value that is large enough to keep up with incoming data
                            # you can even change it to ser.read(2) if you adjust time.sleep to 0.01
                            # ponyframe sorts out the different frames for you, so reading multiple
                            # frames at once isn't an issue
                            line = ser.read(256)
                            if line!= b'':
                                self.tf.accept(line)
                                timeout = time.time()
                            else:
                                if time.time()>timeout+5:
                                    print('next com port')
                                    break
                            time.sleep(.1)
                except (serial.serialutil.PortNotOpenError, 
                        serial.serialutil.SerialException) as e:
                    print(e)
                    pass
                time.sleep(2)
            print("Tried all com ports, starting over")
    
    def error_listener(self, tf, frame):
        print(f'error listener received {frame.data}')
        # acknowledge that we received the error
        self.tf.send(TYPE_ERROR,id=frame.id )
    
    def status_listener(self, tf, frame):
        inbound_message=json.loads(frame.data)
        print(inbound_message))
    
    def id_listener(self, tf, frame):
        print(f'received acknowledgement of frame {frame.id}'

if __name__ == '__main__':
    example = PonyFrame_example()
    example.send_receive()
```
