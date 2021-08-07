# USB driver for linux

First part displays info about the pen drive on the kernel log. The second part will read and write to the USB device via the device file.

I have used Loadable kernel module (LKM). LKMs are pieces of kernel code which are not complied into in the kernel, but they are instead linked to the kernel once they are loaded. When the module is loaded it prints to log file letting you know that the model was either successfully or unsuccessfully loaded into the kernel. Once successfully loaded the module then registers a device driver and a listener to tell if a USB device is plugged into a USB port. When a device is plugged in to the port, the module registers the device, also giving the device its ID, and prints to the log file that a device has been plugged in. After the device is removed from the port, the module attempts to unregister the device, then writes to log file if the device was unregistered successfully or not. The log file is located in var/log/message folder.

The most basic form of USB communication is through something called an endpoint. The below figure shows a typical snippet of USB device specifications for devices connected on a system. To be specific, the “E: ” lines in the figure shows example of an interrupt endpoint of a UHCI Host Controller and two bulk endpoints of the pen drive under consideration. Also, the endpoint numbers (in hex) are respectively 0x81, 0x02. The MSB of the first and third being 1 indicating “in” endpoints, represented by “(I)” in the figure. Second one is an “(O)” for the “out” endpoint. 
<details>
<summary>get your pendrive specifications (product id, vendor id and bulk endpoints here)</summary>
<img src="https://user-images.githubusercontent.com/84666741/128592960-55be48e5-1b65-4157-893a-5bf468b0bcba.png">
</details>
USB endpoints are bundled up into interfaces. Because a USB interface represents basic functionality, each USB driver controls an interface
USB interfaces are described in the kernel with the struct usb_interface structure. This structure is what the USB core passes to USB drivers and is what the USB driver then is in charge of controlling.

## Hardware/Software used : 
  -	ORACLE VM VirtualBox
  -	Ubuntu 20.04
  -	USB device

<details>
<summary>Code Explanation</summary>
<br>
In USB driver the fields to be provided are the driver’s name, ID table for auto-detecting the particular device and the 2 callback functions to be invoked by USB core during hot-plugging and hot-removal of the device, respectively. 

We need the constructor and the destructor.
The USB core APIs for the same are as follows (prototyped in `<linux/usb.h>`):
```
int usb_register(struct usb_driver *driver);
void usb_deregister(struct usb_driver *);
```

the probe & disconnect callbacks, which are invoked by USB core for every interface of the registered device, have the corresponding interface handle as their first parameter. Refer the prototypes below:
```
int (*probe)(struct usb_interface *interface, const struct usb_device_id *id);
void (*disconnect)(struct usb_interface *interface);
```

So with the interface pointer, all information about the corresponding interface can be accessed. And to get the container device handle, the following macro is used:
```
struct usb_device device = interface_to_usbdev(interface);
```

Adding these to the program we get code for `pen_info.c` 

Moreover, as the file operations (write, read, …) are now provided, that is where exactly we need to do the data transfers to and from the USB device. So, `pen_write()` and `pen_read()`below shows the possible calls to `usb_bulk_msg()` (prototyped in `<linux/usb.h>`) to do the transfers over the pen drive’s bulk end points 0x81 and 0x02, respectively.

Note that a pen drive belongs to a USB mass storage class, which expects a set of SCSI like commands to be transacted over the bulk endpoints. So, a raw read/write as shown in the code listing below may not really do a data transfer as expected, unless the data is appropriately formatted. But still, this summarizes the overall code flow of a USB driver.
</details>

## Execution
### Building The Module
```
make
```
The module is compiled to .ko file

The driver was loaded  using `insmod pen_driver.ko`. The pen drive was plugged in (after making sure that the usb-storage driver is not already loaded). Checking for the dynamic creation of `/dev/pen0`(0 being the minor number obtained – check dmesg logs for the value on your system) was implemented. `write/read` operations were performed on `/dev/pen0`. The desired output was seen. And hence, the pen drive was plugged out and the driver was unloaded  using `rmmod pen_driver`.
<details>
<summary>Screenshots</summary>
<br>
<h3>Part 1</h3> 
pen_info.ko
<br>
<img src="https://user-images.githubusercontent.com/84666741/128593850-867ccf4e-2b5d-409c-86b6-a90233e69502.png">
<img src="https://user-images.githubusercontent.com/84666741/128593890-db229157-9bf1-4668-8ed4-39277a89c222.png">

<h3>Part 2</h3>
pen_driver.ko
<br>
<img src="https://user-images.githubusercontent.com/84666741/128593604-f9bd13bb-94af-4526-b303-8bb4461578f4.png">

Dynamic creation of pen0
<br>
<img src="https://user-images.githubusercontent.com/84666741/128593664-ed54d8da-5f5c-4aaa-a416-623508db938e.png">

Read/write to pen0
<br>
![image](https://user-images.githubusercontent.com/84666741/128593909-5702da37-dc88-4074-bb93-c63ed4551cb0.png)
</details>

<h3>Tips</h3>
Show loaded modules

```
lsmod
```
show the kernel log

```
dmesg
```
show info about module

```
modinfo pen_driver.ko
```

## Conclusion 
Thought out this project we learned that loading LKMs while the kernel is running is more efficient for modules that are not used on a normal basis. This way the module does not have to be manually loaded every power up. Depending on the functionality and the number of times it may be needed should determine whether to compile the module with the kernel or to load it.
The significant advantages of LKMs have proven itself to be desired and many operating systems support them, although may be called something different depending on the system. Although they may not be the best solution in every case, it is a very popular technic. Most device driver use LKMs to tell the controller how to communicate and interact with the device.

*This project was done as a learning excercise*
## Reference
 The project is based on :
 https://sysplay.github.io/books/LinuxDrivers/book/index.html
 