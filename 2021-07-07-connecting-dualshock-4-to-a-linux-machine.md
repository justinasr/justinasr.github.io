# Connecting DualShock 4 to a Linux machine

The first and probably the most obvious task when dealing with bluetooth is to perform a scan of nearby devices. Fortunately Linux already comes with a bluetooth stack and that are going to make this difficult mess a little bit easier. So where does one start in writing a bluetooth scanner?

First of all, what do scanners do? At a high level, they return a list of devices, or, to be more precise, device addresses and device names. Names are human readable descriptions for identification and addresses are unique IDs are used by devices for connecting and keeping the connection.

## Bluetooth Device Address

Bluetooth device address (BD_ADDR) is 48 bits – 6 bytes that are unique for each device. It is like a MAC address for a network card and 48 bits mean that there can be up to 248 unique combinations. First 24 bits – 3 bytes uniquely identify device manufacturer and other 3 bytes are assigned by that manufacturer, [I guess] similar to a serial number. Such details are not important at the moment, we only need to know that it is 6 bytes and when they are represented as a sting, they are usually written like 6 hexadecimal values separated by columns, e.g. 01:23:45:67:89:AB. Conveniently, there is a already available typedef struct for such address – bdaddr_t. If we look at the Linux kernel and replace __u8 with uint8_t, it looks like this:

```c
/* BD Address */
typedef struct {
    uint8_t b[6];
} __packed bdaddr_t;
```
6 unsigned one-byte integers, i.e. 0 – 255 or 0x00 – 0xFF.

There are two functions that help to convert strings to bdaddr_t and back:

```c
int str2ba(const char *str, bdaddr_t *ba)
int ba2str(const bdaddr_t *ba, char *str)
```
On success they return 0, otherwise a negative value. These functions are pretty straightforward and their [implementations are no more than a few lines](https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/lib/bluetooth.c).

## Bluetooth Adapter

Second boilerplate step is to find a bluetooth adapter that we could use to do the scan. hci_get_route finds and returns a device ID for given bluetooth address:

```c
int hci_get_route(bdaddr_t *bdaddr)
```
If bdaddr is NULL, hci_get_route returns first bluetooth adapter’s ID:

```c
int adapterDeviceID = hci_get_route(NULL);
```
To get a list of all bluetooth adapters on your machine, use

```bash
hcitool dev
```
And convert string to bdaddr_t with str2ba. Anyway, for now, NULL is good enough if there is only one bluetooth adapter.

At this point we can also open a socket to the bluetooth adapter. This socket will be used to get the bluetooth device name. It is not mandatory, but very nice to see not only device address, but human readable name too. For this we will use hci_open_dev that takes in device ID and returns a socket:

```c
int hci_open_dev(int dev_id)
```
Pass in bluetooth adapter’s ID that we got in the first part of this section:

```c
int adapterSocket = hci_open_dev(adapterDeviceID);
```
And we have a socket to the bluetooth adapter as well as adapter’s device ID.

## Inquiry

We know what we want, we have a bluetooth adapter, now we need to find a way to get what we want. To do this, we have to do an inquiry. According to Google,

> Inquiry – an act of asking for information.

As usual with C programming, when calling a function, it’ll take in a pointer to a data structure that will be filled with data. This struct is called inquiry_info and can be found in hci.h:

```c
struct inquiry_info {
    bdaddr_t bdaddr;
    uint8_t  pscan_rep_mode;
    uint8_t  pscan_period_mode;
    uint8_t  pscan_mode;
    uint8_t  dev_class[3];
    __le16   clock_offset;
} __packed;
```
[Only important variable](https://people.csail.mit.edu/albert/bluez-intro/c404.html) is the first one – bdaddr which is the device address. To perform an inquiry, once again, there is a function for that:

```c
int hci_inquiry(int dev_id,
                int len,
                int max_rsp,
                const uint8_t *lap,
                inquiry_info **ii,
                long flags);
```

Now this function takes quite a few arguments. First argument is device ID – this is an bluetooth adapter’s ID that will be used for inquiry. Other arguments are very well explained [here](https://people.csail.mit.edu/albert/bluez-intro/c404.html) (I prefer to use more descriptive variable names that I indicated in square brackets):

> The inquiry lasts for at most 1.28 * len [scanLength] seconds, and at most max_rsp [scanMaxDevices] devices will be returned in the output parameter ii [inquiryInfos], which must be large enough to accommodate max_rsp [scanMaxDevices] results. We suggest using a max_rsp [scanMaxDevices] of 255 for a standard 10.24 second inquiry. If flags is set to IREQ_CACHE_FLUSH, then the cache of previously detected devices is flushed before performing the current inquiry. Otherwise, if flags is set to 0, then the results of previous inquiries may be returned, even if the devices aren't in range anymore.

hci_inquiry returns a non-negative number that shows how many devices were found. If there was an error, it returns a negative number.

## Scanning

If we combine everything that is mentioned above, we get a code that looks like this:

```c
// Get adapter device ID
int adapterDeviceID = hci_get_route(NULL);
// Open socket to bluetooth adapter
int adapterSocket = hci_open_dev(adapterDeviceID);
// Scan duration is scanLength * 1.28 seconds, 8 * 1.28 = 10.24s
int scanLength = 8;
// Maximum number of devices to find
int scanMaxDevices = 255;
// Allocate memory for scanMaxDevices number of inquiry_info structs
inquiry_info* inquiryInfos = (inquiry_info*)malloc(scanMaxDevices * sizeof(inquiry_info));
// Perform the inquiry - "scan"
int foundDevices = hci_inquiry(adapterDeviceID, scanLength, scanMaxDevices, NULL, &inquiryInfos, IREQ_CACHE_FLUSH);
// Iterate through found devices and print their bluetooth addresses
for (int i = 0; i < foundDevices; i++) {
    bdaddr_t deviceAddress = (inquiryInfos + i)->bdaddr;
    char deviceAddressString[19] = {0};
    ba2str(&deviceAddress, deviceAddressString);
    printf("Address: %s\n", deviceAddressString);
}
```
This program prints a list of bluetooth devices that adapter was able to find. As far as I understand, bluetooth devices do not broadcast their name, you have to explicitly ask for it. This is where adapterSocket will become handy. Function hci_read_remote_name can get a device name (string) for a device if you know it’s device address:

```c
int hci_read_remote_name(int sock,
                         const bdaddr_t *ba,
                         int len, 
                         char *name,
                         int timeout)
```
hci_read_remote_name returns 0 on success and -1 on failure and copies human readable name, at most len characters, to given char array. According to bluetooth specification, names can be up to 248 bytes long. Timeout is in milliseconds. We update our for loop after the scan to not only show device addresses, but their names too:

```c
for (int i = 0; i < foundDevices; i++) {
    bdaddr_t deviceAddress = (inquiryInfos + i)->bdaddr;
    char deviceAddressString[19] = {0};
    char deviceName[248] = {0};
    ba2str(&deviceAddress, deviceAddressString);
    hci_read_remote_name(adapterSocket, &deviceAddress, sizeof(deviceName), deviceName, 0);
    printf("Address: %s Name: %s\n", deviceAddressString, deviceName);
}
```
Finally, a scanner that prints a list of device addresses and names. Full source code:

```c
#include <stdlib.h>
#include <bluetooth/bluetooth.h>
#include <bluetooth/hci.h>
#include <bluetooth/hci_lib.h>
 
// Compile with gcc -o scan scan.c -lbluetooth
int main(int argc, char **argv) {
    // Get adapter device ID
    int adapterDeviceID = hci_get_route(NULL);
    // Open socket to bluetooth adapter
    int adapterSocket = hci_open_dev(adapterDeviceID);
    // Scan duration is scanLength * 1.28 seconds, 8 * 1.28 = 10.24s
    int scanLength = 8;
    // Maximum number of devices to find
    int scanMaxDevices = 255;
    // Allocate memory for scanMaxDevices number of inquiry_info structs
    inquiry_info* inquiryInfos = (inquiry_info*)malloc(scanMaxDevices * sizeof(inquiry_info));
    // Perform the inquiry - "scan"
    int foundDevices = hci_inquiry(adapterDeviceID, scanLength, scanMaxDevices, NULL, &inquiryInfos, IREQ_CACHE_FLUSH);
    // Iterate through found devices and print their bluetooth addresses
    for (int i = 0; i < foundDevices; i++) {
        bdaddr_t deviceAddress = (inquiryInfos + i)->bdaddr;
        char deviceAddressString[19] = {0};
        char deviceName[248] = {0};
        ba2str(&deviceAddress, deviceAddressString);
        hci_read_remote_name(adapterSocket, &deviceAddress, sizeof(deviceName), deviceName, 0);
        printf("Address: %s Name: %s\n", deviceAddressString, deviceName);
    }
}
```

Compile with gcc:

```bash
gcc -o blue blue.c -lbluetooth
```
Run:

```bash
./blue
```
Of course, this code does not check for errors and works only in ideal conditions. In the real world, any of these functions could return an error that should be gracefully handled.

Once we know how to scan for devices around us, let’s try to connect to one. As name of this post suggests it is going to be a DualShock 4 game controller.

## Preparing to connect

First of all, device address string has to be converted to a bdaddr_t. I will omit scanning part and hardcode my controller’s address just to save some time while debugging. This time I will also include return value checking to catch any mishaps.

```c
// Hardcoded device address
char dest[18] = "01:23:45:67:89:AB";
bdaddr_t address;
// Convert char array to bdaddr_t
if (str2ba(dest, &address) != 0) {
    printf("Could not convert device address %s\n", dest);
}
```
Now we have to create a struct that will have all the connection details. Struct type is sockaddr_l2 and it looks like this:

```c
// L2CAP socket address
struct sockaddr_l2 {
    sa_family_t    l2_family;
    unsigned short l2_psm;
    bdaddr_t       l2_bdaddr;
    unsigned short l2_cid;
    uint8_t        l2_bdaddr_type;
};
```
We will have to set three attributes – l2_bdaddr, l2_family and l2_psm. l2_bdaddr will be the device address that we just converted from a string. l2_family is the address family which will, in this case, be a constant AF_BLUETOOTH, that has a value of 31. Other well know address family types are AF_INET and AF_INET6 (values 2 and 10, IPv4 and IPv6 respectively). The last thing, l2_psm will be set to 0x13. This means that this is a HID (Human interface device) and that we will “listen on interrupt channel”, basically whenever something will change on the remote controller’s end, it’ll send us a message with updated information, such as button press. Magic value 0x13 comes from this specification.

```c
struct sockaddr_l2 interruptSocketAddress = { 0 };
interruptSocketAddress.l2_bdaddr = address;
interruptSocketAddress.l2_family = AF_BLUETOOTH;
interruptSocketAddress.l2_psm = htobs(0x13);
```
Function htobs just converts 0x13 value to have a correct endianness.

## Connect!

Finally we arrive at the pinnacle of this post – connecting to the DS4 controller.

We just need to create a socket:

```c
int interruptSocket;
interruptSocket = socket(AF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_L2CAP);
```
AF_BLUETOOTH is the address family, SOCK_SEQPACKET is socket type and BTPROTO_L2CAP is telling to use L2CAP protocol stack. I have not fully figured out what SOCK_SEQPACKET and L2CAP exactly are, but I found this for the former:

> SOCK_SEQPACKET - Provides a sequenced, reliable, two-way connection- based data transmission path for datagrams of fixed maximum length; a consumer is required to read an entire packet with each input system call.BTPROTO_L2CAP
Then we connect to the socket using socket address struct:

```c
int interruptSocketStatus;
interruptSocketStatus = connect(interruptSocket,
                                (struct sockaddr *)&interruptSocketAddress,
                                sizeof(interruptSocketAddress));
if (interruptSocketStatus == 0) {
    printf("Connected interrupt socket\n");
} else {
    perror("Something went wrong");
}
```
Basically it takes a socket and connects it using the given address struct. Now interrupt socket can be used to send and receive data. However, since this is interrupt socket, we’ll only read data from it. Writing is useless as no one is listening to this socket on the other side.

## Reading DS4 messages

Reading a socket is pretty easy. We know that DS4 sends 78 bytes through the interrupt socket:

```c
unsigned char resp[78];
while (1) {
    recv(interruptSocket, resp, sizeof(resp), 0);
}
```
This will continuously read interrupt socket and put read data into resp array. If we write little helper function to print char array as hexadecimal values, we can see the full message in hex:

```c
void printHex(char* chars, int length) {
    for (int i = 0; i < length; i++) {
        printf("%02hhX ", chars[i]);
    }
    printf("\n");
}
```
Pass the resp array to printHex function:

```c
while (1) {
    recv(interruptSocket, resp, sizeof(resp), 0);
    printHex(resp, sizeof(resp));
}
```
Now you should see a lot of hex values being printed on a screen. Some of them are fluctuating, some change only when a button is pressed. Next step will be understanding these values and printing DS4 actions in human readable form like “Pressed triangle”.

Full source code:
```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <bluetooth/bluetooth.h>
#include <bluetooth/l2cap.h>
 
void printHex(char* chars, int length) {
    for (int i = 0; i < length; i++) {
        printf("%02hhX ", chars[i]);
    }
    printf("\n");
}
 
int main(int argc, char **argv) {
    // Hardcoded device address
    char dest[18] = "DC:0C:2D:B8:2B:9A";
    bdaddr_t address;
    // Convert char array to bdaddr_t
    if (str2ba(dest, &address) != 0) {
        printf("Could not convert device address %s\n", dest);
    }
    struct sockaddr_l2 interruptSocketAddress = { 0 };
    interruptSocketAddress.l2_bdaddr = address;
    interruptSocketAddress.l2_family = AF_BLUETOOTH;
    interruptSocketAddress.l2_psm = htobs(0x13);
 
    // Create a socket for and for interrupts
    int interruptSocket;
    interruptSocket = socket(AF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_L2CAP);
 
    // Connect these sockets to the device
    int interruptSocketStatus;
    interruptSocketStatus = connect(interruptSocket,
                                    (struct sockaddr *)&interruptSocketAddress,
                                    sizeof(interruptSocketAddress));
    if (interruptSocketStatus == 0) {
        printf("Connected interrupt socket\n");
    } else {
        perror("Something went wrong");
    }
 
    unsigned char resp[79];
    while (1) {
        recv(interruptSocket, resp, sizeof(resp), 0);
        printHex(resp, sizeof(resp));
    }
}
```
Compile with gcc:

```bash
gcc -o blue blue.c -lbluetooth
```

Run:

```bash
./blue
```

References:

https://github.com/torvalds/linux/blob/master/include/net/bluetooth/bluetooth.h
https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/lib/bluetooth.c
https://github.com/torvalds/linux/blob/master/include/net/bluetooth/hci.h
https://people.csail.mit.edu/albert/bluez-intro/c404.html
https://macaddresschanger.com/what-is-bluetooth-address-BD_ADDR

https://man7.org/linux/man-pages/man2/connect.2.html
https://man7.org/linux/man-pages/man2/socket.2.html
https://www.bluetooth.com/specifications/assigned-numbers/logical-link-control/
