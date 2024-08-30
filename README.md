# PonyFrame

This is based on [PonyFrame](https://github.com/MightyPork/PonyFrame), but adapted for use with asyncio

PonyFrame itself is based on [TinyFrame](https://github.com/MightyPork/TinyFrame). Overall there might be missing pieces and holes, but it gets at least some jobs done.

This implements error checking for serial communications. 

Usage, tested on circuitpython on the Xiao RP2040:

first enable serial communications, not just the interactive terminal. Add the following to boot.py
```
import usb_cdc
usb_cdc.enable(console=True, data=True)
```

```
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
        await asyncio.sleep(0.5)
        if not self.error_status:
            if self.data_type == 'sin':
                value = {'sine data' : math.sin(n/100)} #read a sensor or something here
                print(value)
            elif self.data_type == 'cos':
                value = {'cosine data' : math.cos(n/100)} #read a different sensor...
                                    #something had to change for the example
            else:
                value = {'error':'invalid data type requested'}
                self.tf.query(self.TYPE_ERROR, error_id_listener, json.dumps(value).encode('UTF-8'))
                # a query expects a response (which should be sent automatically by tinyframe
                self.error_status = True
                break
            self.tf.send(self.TYPE_STATUS, json.dumps(value).encode('UTF-8'))
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

start of computer program: 

TYPE_RESPONSE = 0x0000
TYPE_STATUS = 0x0001
TYPE_INSTRUCTION = 0x0002
TYPE_ERROR = 0x0003 

class aioPonyFrame_example():
  def __init__(self):
      ports = serial.tools.list_ports.comports()
  def send_receive(self):
    ports
    with serial.Serial(port=current_port[0],
      baudrate=19600,
      parity=serial.PARITY_NONE,
      stopbits=serial.STOPBITS_ONE,
      bytesize=serial.EIGHTBITS) as ser:
      ser.write_timeout=0
      ser.timeout = 0
      tf=TinyFrame.TinyFrame()
      tf.TYPE_BYTES = 0x02
      tf.CKSUM_TYPE = 'crc16'
      tf.SOF_BYTE = 0x55
      tf.write = ser.write
      tf.add_fallback_listener(self.fallback_listener)
      tf.add_type_listener(TYPE_DRIVER_STATUS, self.driver_status_listener)
      tf.add_type_listener(TYPE_HEATER_STATUS, self.heater_status_listener)
