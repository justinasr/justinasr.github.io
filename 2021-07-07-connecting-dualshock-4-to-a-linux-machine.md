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

References:

https://github.com/torvalds/linux/blob/master/include/net/bluetooth/bluetooth.h
https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/lib/bluetooth.c
https://github.com/torvalds/linux/blob/master/include/net/bluetooth/hci.h
https://people.csail.mit.edu/albert/bluez-intro/c404.html
https://macaddresschanger.com/what-is-bluetooth-address-BD_ADDR
