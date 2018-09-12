# Programming Guide

Virtual channels are referred to by a seven-character (or shorter) ASCII
name. In several previous versions of the ICA protocol, virtual channels
were numbered; the numbers are now assigned dynamically based on the
ASCII name, making implementation easier.

When developing virtual channel code for internal use only, you can use
any seven-character name that does not conflict with existing virtual
channels. Use only upper and lowercase ASCII letters and numbers. Follow
the existing naming convention when adding your own virtual channels.

The predefined channels, which begin with the OEM identifier CTX, are
for use only by Citrix.

## Design Suggestions

Follow these suggestions to make your virtual channels easier to design and enhance:

-  When you design your own virtual channel protocol, allow for the flexibility to add features. Virtual channels have version numbers that are exchanged during initialization so that both the client and the server detect the maximum level of functionality that can be used.

For example, if the client is at
Version 3 and the server is at Version 5, the server does not send any packets with functionality beyond Version 3 because the client does not know how to interpret the newer packets.

-  Because the server side of a virtual channel protocol can be implemented as a separate process, it is easier to write code that interfaces with the Citrix-provided virtual channel support on the server than on the client (where the code must fit into an existing code structure). The server side of a virtual channel simply opens the channel, reads from and writes to it, and closes it when done. Writing code for the server-side is similar to writing an application, which uses services exported by the system. It is easier to write an application to handle the virtual channel communication because it can then be run once for each ICA connection supporting the virtual channel.

Writing for the client-side is similar to writing a driver, which must provide services to the system in addition to using system services. If a service is written, it must manage multiple connections.

-  If you are designing new hardware for use with new virtual channels (for example, an improved compressed video format), make sure the hardware can be detected so that the client can determine whether or not it is installed.

Then the client can communicate to the server if the hardware is available before the server uses the new data format. Optionally, you could have the virtual driver translate the new data format for use with older hardware.

-  There might be limitations preventing your new virtual channel from performing at an optimum level. If the client is connecting to the server running Citrix Virtual Apps and Desktops through a modem or serial connection, the bandwidth might not be great enough to properly support audio or video data. You can make your protocol adaptive, so that as bandwidth decreases, performance degrades gracefully, possibly by sending sound normally but reducing the frame rate of the video to fit the available bandwidth.

-  To identify where problems are occurring (connection, implementation, or protocol), first get the connection and communication working. Then, after the virtual channel is complete and debugged, do some time trials and record the results.These results establish a baseline for measuring further optimizations such as compression and other enhancements so that the channel requires less bandwidth.

-  The time stamp in the pVdPoll variable can be helpful for resolving timing issues in your virtual driver. It is a ULONG containing the current time in milliseconds. The pVdPoll variable is a pointer to a DLLPOLL structure. See Dllapi.h (in
src/inc/) for definitions of these structures.

## Server-Side Functions Overview

Server-side functions are entry points to virtual channel services
provided by the ICAsubsystem on the Citrix Virtual Apps and Desktops server. Wfapi.h
contains constants and function prototypes.

Use these functions to open and close virtual channels and to read,
write, query, and purge incoming or outgoing data.

The words IN and OUT in the function calling conventions are for
clarification only. They are defined as blank in Windef.h. If you do not
have access to Windef.h, add the following to a header file for your
project:

```
#ifndef IN
#define IN
#endif
#ifndef OUT
#endif
```

| Function | Description|
|----------|------------|
|WFVirtualChannelClose | Closes an open virtual channel handle. |
|WFVirtualChannelOpen | Opens a handle to a specific virtual channel.|
|WFVirtualChannelPurgeInput|Purges all queued input data sent from the client to the server on a specific virtual channel. |
|WFVirtualChannelPurgeOutput |Purges all queued output data sent from the server to the client on a specific virtual channel.|
|WFVirtualChannelQuery |Returns data related to a virtual channel. |
|WFVirtualChannelRead  | Reads data from a virtual channel.|
|WFVirtualChannelWrite | Writes data to a virtual channel.|

## Client-Side Functions Overview

The client software is built on a modular configurable architecture that
allows replaceable, configurable modules (such as virtual channel
drivers) to handle various aspects of an ICA connection. These modules
are specially formatted and dynamically loadable. To accomplish this
modular capability, each module (including virtual channel drivers)
implements a fixed set of function entry points.

There are three groups of functions: user-defined, virtual driver
helper, and memory INI.

### User-defined Functions

To make writing virtual channels easier, dynamic loading is handled by
the WinStation driver, which in turn calls user-defined functions. This
simplifies creating the virtual channel because all you have to do is
fill in the functions and link your virtual channel driver with
Vdapi.lib (provided with this SDK).

| Function | Description|
|----------|------------|
| DriverClose |Frees private driver data. Called before unloading a virtual driver (generally upon client exit). |
|DriverGetLastError |Returns the last error set by the virtual driver. Not used; links with the common front end, VDAPI.|
|DriverInfo|Retrieves information about the virtual driver.|
|DriverOpen|Performs all initialization for the virtual driver. Called once when the client loads the virtual driver (at startup).|
|DriverPoll|Allows driver to check timers and other state information, sends queued data to the server, and performs any other required processing. Called periodically to see if the virtual driver has any data to write.|
|DriverQueryInformation |Retrieves run-time information from the virtual driver.|
|DriverSetInformation|Sets run-time information in the virtual driver.|
|ICADataArrival|Indicates that data was delivered. Called when data arrives on the virtual channel.|

### Virtual Driver Helper Functions

The virtual driver uses helper functions to send data and manage the
virtual channel. When the WinStation driver initializes the virtual
driver, the WinStation driver passes pointers to the helper functions
and the virtual driver passes pointers to the user-defined functions.

VdCallWd is linked in as part of VDAPI and is available in all
user-implemented functions. The others are obtained during DriverOpen
when VdCallWd is called with the WDxSETINFORMATION parameter.

Virtual channel drivers can send data from private buffers via the
SendData or QueueVirtualWrite functions obtained during DriverOpen.
Either of these functions may decline to accept the data if the
WinSation Driver itself cannot buffer it. The channel will then need to
retry the send operation on the next DriverPoll.

| Function | Description|
|----------|------------|
|SendData|To send a packet of channel protocol to the server, with a notification option.|
|QueueVirtualWrite|To send a packet of channel protocol to the server. This is a legacy function. Use SendData above for new virtual drivers.|
|VdCallWd|Used to query and set information from the WinStation driver (WD).|

### Memory INI Functions

Memory INI functions read data that the client engine reads from the
Configuration Storage in the registry and stored in a memory INI
structure. These functions must be used because some client devices
store this information in ROM, and only the engine has access to this
INI data. The Memory INI functions read values from this Memory INI
structure as if the values are being read from the Configuration
Storage. The Configuration Storage for specifying virtual channels is in
Software\Citrix\ICA Client\Configuration\Advanced\Modules\.

Important: Access to configuration data might be limited depending on
security restrictions . In particular, the virtual channel might not
have access to all contents of the ICA file. This is controlled by
registry keys in HKEY_LOCAL_MACHINE\SOFTWARE\Citrix\ICA
Client\Engine\Lock down Profiles\All Regions\Lock down. You can use
Memory INI functions read data from the client engine configuration
files stored in both the client installation directory for system wide
settings and $HOME/.ICAClient for user specific settings.

For each entry in appsrv.ini and wfclient.ini, there must be a
corresponding entry in All_Regions.ini for the setting to take effect.
For more information, refer to All_Regions.ini file in the
$ICAROOT/config directory.

| Function | Description |
|----------|-------------|
|miGetPrivateProfileBool |Returns a boolean value.|
|miGetPrivateProfileInt |Returns an integer value. |
| miGetPrivateProfileLong| Returns a long value.|
|miGetPrivateProfileString| Returns a string value.|