## Update: 10 November 2011 ##

---

So upon my endeavor of implementing Qt as a GUI I determined it would be a lot easier and more efficient to use the Windows API. It may perhaps be more tedious to have the GUI elements how you want, but it just feels more streamline and faster. Some of the major things I finally got working is the function that listens to the controller and captures the reports is in its own thread. The code is in no way complete I just figured I'd post updates as they come along. If anyone has any interests or idea by all means let me know. My next step is to implement the ability for [sixutility](http://openaxis.googlecode.com/files/sixutilityGUI.rar) to work either via USB or Bluetooth.

## Udate: 04 November 2011 ##

---

So I got bored with this project for a while and didn't mess with it until recently. I've basically started a utility project (sixutils) that will be the base for doing many things with the ps3 sixaxis controller. As of now things are pretty basic. There is the ability to read the controller's bluetooth address, the controller's master bluetooth address, and get the state of the controller (the HID report that says what keys are pressed). Right now the project is archived and in the downloads section. I will get the source posted soon. Ultimately I would like to make this into a GUI. I was thinking of using Qt to do so, but I'm not very experienced with it. If anybody has any ideas or suggestions for improvement by all means let me know!

## About ##

---

**openaxis** is a project I started to demonstrates the controlling interface between the ps3 sixaxis controler and a PC. The communication is based off a specific USB Request Block (URB) that requests the controller to send the state of itself (i.e. button(s) pressed, analog joystick positions, etc) to the computer which then interprets the 49 byte block into its corresponding parts. It can read everything from button presses, to values of the right and left buttons, to position of joysticks.

## The reverse engineering process ##

---


I know its been done already, but as hard as I looked I couldn't find any kind of source code for the ps3sixaxis\_en program that lets you use the controller on windows. So I spent the day messing around with USBTrace, GlovePIE, and the controller. So it begins...

I used GlovePIE (for no particular reason) to find the USB Request Block that gets the status of the controller.

![http://openaxis.googlecode.com/files/USBTraceLog.png](http://openaxis.googlecode.com/files/USBTraceLog.png)


After I found the URB that gets the controller I used the python script from DIY Kinect Hacking tutorial

![http://openaxis.googlecode.com/files/python_script.png](http://openaxis.googlecode.com/files/python_script.png)

I'm not going to go into detail about how to set everything up (i.e. pyusb, libusb, etc). The DIY Kinect Hacking article is awesome and includes all the preliminary steps.

So after setting up the script and running it all I had to do was press the buttons and watch the screen and analyze what changed. Here is an example of pressing the start button.

![http://openaxis.googlecode.com/files/analysis.png](http://openaxis.googlecode.com/files/analysis.png)

Here you can also see the joystick X and Y values. when you move a joystick all the way left the X value equals 0 and all the way right it equals 255. Same goes for the Y values. up=0, down=255

So with the bit of knowledge I learned today I made a few modification to the OpenKinect c# code. All i have it doing is printing some of the buttons to the console window when you press them on the controller.

![http://openaxis.googlecode.com/files/poc.output.png](http://openaxis.googlecode.com/files/poc.output.png)

> I used SharpDevelop but visual studio should open the solution fine.

## The Protocol ##

---

The computer needs to send a URB (USB Request Block) to the controller telling it to send some data back to the computer. There are multiple URB's used to communicate with the controller however the only one I've found so far is the URB to retrieve the status of the controller.

So when we want to get the status we send a control message. This is done by first building a setup packet `UsbSetupPacket setup = new UsbSetupPacket(0xa1, 0x1, 0x0101, 0, 0x31);` This setup packet is essentially the command we are sending to the controller. It contains the following:
  * bmRequestType - 0xa1 (DEVICE\_TO\_HOST|CLASS|INTERFACE)
  * bRequest - 0x01 (The request type)
  * wValue - 0x0101 (The request value)
  * wIndex - 0x0
  * wLength - 0x31 (There will be 49 bytes transfered by the device)

Once we build the setup packet we send it to the controller as a control transfer: `MyUsbDevice.ControlTransfer(ref setup, buf, (ushort)buf.Length, out len);`
Here `setup` is the setup packet we just created, `buf` is a 49 byte buffer used to store the packet the controller will send containing the status of the controller, and 'len' is just an integer for for how many bytes were actually received.

After the controller sends its 49 byte payload to the computer we have to examine the buffer for the state and values of the buttons or joystick positions.

This is the structure of the payload:
```
struct CONTROLLER_DATA
{
    ushort ????
    byte DIRECTION_PAD
    byte BUTTON_PAD
    byte PS_POWER_BUTTON    //0-depressed; 1-pressed
    byte ??
    JOYSTICK_POSITION LEFT_JOYSTICK
    JOYSTICK_POSITION RIGHT_JOYSTICK
    ushort ????
    ushort ????
    ushort ????
    ushort ??
    byte BTN_L2_VALUE        //0-depressed; 255-fully pressed
    byte BTN_R2_VALUE        //0-depressed; 255-fully pressed
    byte BTN_L1_VALUE        //0-depressed; 255-fully pressed
    byte BTN_R1_VALUE        //0-depressed; 255-fully pressed
    ushort ????
    ushort ????
    ushort ????
    byte ??
    byte ??    //seems to always be 3
    byte ??    //seems to always be 239
    byte MOTOR1
    ushort ????
    ushort ????
    byte ??    //seems to always be 51
    byte ??    //seems to always be 60
    byte ??    //does change. possibly motor?
    byte ??    //seems to always be 1
    byte MOTOR2
    byte HIT    //Not for sure but appears to be 1 normally, then becomes 2 if the controller is hit
    byte ROLL    //0-centered; left-positive; right-negative
    byte ??      //seems to always be 2
    byte PITCH   //0-centered; up-positive; down-negative
    byte ??      //seems to always be 1
    byte Z_AXIS  //Not sure what axis this is. It does change with movement tho
    byte ??      //seems to always be 1
    byte ??      //accelerometer perhaps? values are positive when you move controller left and negative when 
                //you move the controller right
};
```

Some of values need to be enumerated. Here is what the values are:

```
enum DIRECTION_PAD : byte
{
    SELECT  = 0x01
    BTN_L3  = 0x02
    BTN_R3  = 0x04
    START   = 0x08
    UP      = 0x10
    RIGHT   = 0x20
    DOWN    = 0x40
    LEFT    = 0x80
};

enum BUTTON_PAD : byte
{
    LEFT2      = 0x01
    RIGHT2     = 0x02
    LEFT1      = 0x04
    RIGHT1     = 0x08    
    TRIANGLE   = 0x10
    CIRCLE     = 0x20
    CROSS      = 0x40
    SQUARE     = 0x80    
};

enum MOTOR1 : byte
{
    ON    = 0x14
    OFF   = 0x16
}

enum MOTOR2 : byte
{
    ON    = 0x40
    OFF   = 0xc0
}

struct JOYSTICK_POSITION
{
    byte X_AXIS        //0-LEFT; 255-RIGHT
    byte Y_AXIS        //0-UP; 255-DOWN
};
```

So once we have all the data in our structure we can just read whatever values we want and it will give us the state of the controller.

## Links ##

---

  * [Awesome tutorial DIY hacking the Kinect](http://http://ladyada.net/learn/diykinect/)
  * [The driver that has to be used with the device](http://www.libusb.org/wiki/windows_backend)
  * [C# library currently used for the implementation of this hack](http://sourceforge.net/projects/libusbdotnet/)
  * [Excellent information about USB](http://www.beyondlogic.org/usbnutshell/usb1.shtml)
  * [SharpDevelop: my prefered C# IDE](http://www.icsharpcode.net/opensource/sd/)