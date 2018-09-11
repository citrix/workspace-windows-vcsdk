# Architecture

A Citrix Independent Computing Architecture (ICA) virtual channel is a
bidirectional error-free connection for the exchange of generalized
packet data between a server running the Citrix component and a client
device. Developers can use virtual channels to add functionality to
clients. Uses for virtual channels include:

*  Support for administrative functions
*  New data streams (audio and video)
*  New devices, such as scanners, card readers, and joysticks)

## Virtual Channel Overview

An ICA virtual channel is a bidirectional error-free connection for the
exchange of generalized packet data between a client and a server
running Citrix Virtual Apps and Desktops. Each implementation of an ICA
virtual channel consists of two components:

*  Server-side portion on the computer running Citrix Virtual Apps and Desktops

 The virtual channel on the server side is a normal Win32 process; it can
be either an application or a Windows NT service.

*  Client- side portion on the client device

 The client-side virtual channel driver is a dynamically loadable module
(.DLL) that executes in the context of the client. You must write your
virtual driver.

This figure illustrates the virtual channel client-server connection:

![alt_text!](./virtual-channel-overview.png)

The `WinStation` driver is responsible for demultiplexing the virtual
channel data from the ICA data stream and routing it to the correct
processing module (in this case, the virtual driver). The `WinStation`
driver is also responsible for gathering and sending virtual channel
data to the server over the ICA connection. On the client side, the
`WinStation` driver is also called the client engine, or simply the
engine.

The following is an overview of client-server data exchange using a
virtual channel:

1.  The client connects to the server running Citrix Virtual Apps and Desktops. The
    client passes information about the virtual channels it supports
    to the server.

2.  The server-side application starts, obtains a handle to the virtual
    channel, and optionally queries for additional information about
    the channel.

3.  The client-side virtual driver and server-side application pass data
    using the following two methods:

*  If the server application has data to send to the client, the data is sent to the client immediately. When the client receives the data, the WinStation driver demultiplexes the virtual channel data from the ICA stream and passes it immediately to the client virtual driver.

*  If the client virtual driver has data to send to the server, the data may be sent immediately, or it may be sent the next time the WinStation driver polls the virtual driver. When the data is received by the server, it is queued until the virtual channel application reads it. There is no way to alert the server virtual channel application that data was received.

4\.  When the server virtual channel application is finished, it closes
    the virtual channel and frees any allocated resources.

## ICA and Virtual Channel Data Packets

Virtual channel data packets are encapsulated in the ICA stream between
the client and the servers. Because ICA is a presentation-level protocol
and runs over several different transports, the virtual channel
application programming interface (API) enables developers to write
their protocols without worrying about the underlying transport. The
data packet is preserved.

For example, if 100 bytes are sent to the server, the same 100 bytes are
received by the server when the virtual channel is demultiplexed from
the ICA data stream. The compiled code runs independently of the
currently configured transport protocol.

The ICA engine provides the following services to the virtual

### channel: Packet encapsulation

ICA virtual channels are packet-based, meaning that if one side performs
a write with a certain amount of data, the other side receives the
entire block of data when it performs a read. This contrasts with TCP,
for example, which is stream-based and requires a higher-level protocol
to parse out packet boundaries. Stated another way, virtual channel
packets are contained within the ICA stream, which is managed separately
by system software.

### Error correction

ICA provides its own reliability mechanisms even when the underlying
transport is unreliable. This guarantees that connections are error free
and that data is received in the order in which it is sent.

### Flow control

The virtual channel API provides several types of flow control. This
allows designers to structure their channels to handle only a specific
amount of data at any one time. See Flow Control for more information.

## Client WinStation Driver and Virtual Driver Interaction

Client virtual drivers may elect to send data in either of two modes:

*  Polling mode
*  Immediate mode

If operating in the polling mode, the WinStation driver polls each
virtual driver regularly by calling its DriverPoll function. When
DriverPoll is called, the virtual driver should immediately check any
necessary state information, send any queued data, and return control to
the WinStation driver.

If operating in the immediate mode, the virtual driver may send data at
any time. For example, suppose the driver receives a packet from the
server in the ICADataArrival function. In the immediate send -data mode,
the driver may send data immediately in response to the packet received,
and then control will return to the WinStation driver from the
ICADataArrival function.

Whenever the virtual driver attempts to send data to the server, it
should be prepared for a data send operation to sometimes be declined.
This may occur because the WinStation Driver supports a reasonable but
not excessive amount of queued backlog waiting to be sent to the server.
If a send operation is declined, the virtual driver must arrange to
retry the send later.

In any case, whether operating in the polling or immediate mode, the
virtual driver must never block. When any of the driver functions are
called by the WinStation driver, the virtual driver must immediately
check any necessary state information, send any queued data, and return
control to the WinStation driver.

The following process occurs when a user starts the client:

1.  At client load time, the client engine reads the Configuration
    Storage in the configuration files to determine the modules to
    configure, including how to configure the virtual channel drivers.

2.  The client engine loads the virtual channel drivers defined in the
    Configuration Storage in the configuration files by calling the
    Load function, which must be exported explicitly by the virtual
    channel driver .Dll. The Load function is defined in the static
    library file Vdapi.lib, which is provided in this SDK. Every
    driver must link with this library file. The Load function
    forwards the driver entry points defined in the .Dll to the
    client engine.

3.  For each virtual channel, the WinStation driver calls the DriverOpen
    function, which establishes and initializes the virtual channel.
    The WinStation driver passes the addresses of the output buffer
    management functions in the WinStation driver to the virtual
    channel driver. The virtual channel driver passes the address of
    the ICADataArrival function to the WinStation driver. The
    WinStation driver calls the DriverOpen function for each virtual
    driver when the client loads, not when the virtual channel is
    opened by the server-side application.

4.  When virtual channel data arrives from the server, the WinStation
    driver calls the ICADataArrival function for that virtual driver.

5.  If you are using the send-data polling mode, the virtual driver
    cannot initiate data transfers. Instead, the WinStation driver
    calls DriverPoll to poll for data to send to the server. To send
    data, the virtual channel driver can use the QueueVirtualWrite
    function (this address is obtained during initialization) to send
    a block of data to the server-side version of the channel. During
    DriverPoll, the virtual driver may try to send one or more
    packets (VirtualWrites) until there is no more data to send, or
    the QueueVirtualWrite function returns the error code
    CLIENT_ERROR_NO_OUTBUF, to indicate that there is no more buffer space, and that the data in question has not been accepted, and must be retried later (normally on
a later DriverPoll.

6.  If using the immediate send-data mode, the virtual driver may send
    data at any time. To send data, the virtual channel driver will
    use the SendData function (this address is obtained
    during initialization) to send a block of data to the server-side
    version of the channel. The virtual driver may try to send one or
    more packets (VirtualWrites) until there is no more data to send,
    or the SendData function returns the error code
    CLIENT_ERROR_NO_OUTBUF, to indicate that there is no more
    buffer space, and that the data in question has not been accepted,
    and must be retried later. The retry will normally happen when the
    WinStation driver presents a special “notification” call to the
    virtual driver’s DriverPoll function. The notification DriverPoll
    call is made when the WinStation driver detects that buffers have
    been freed and the send data operation may be retried.

## Module.ini

The Citrix Workspace app for Windows use settings stored in Module.ini to determine which
virtual channels to load. Driver developers can also use Module.ini to
store parameters for virtual channels. Module.ini changes are effective
only before the installation. After the installation, you must modify
the Configuration Storage in the configuration files to add or remove
virtual channels.

Use the memory INI functions to read data from Configuration Storage.

## Virtual Channel Packets

ICA does not define the contents of a virtual channel packet. The
contents are specific to the particular virtual channel and are not
interpreted or managed by the ICA data stream manager. You must develop
your own protocol for the virtual channel data.

A virtual channel packet can be any length up to the maximum size
supported by the ICA connection. This size is independent of size
restrictions on the lower-layer transport. These restrictions affect the
server- side WFVirtualChannelRead and WFVirtualChannelWrite functions
and the QueueVirtualWrite and SendData functions on the client side. The
maximum packet size is 5000 bytes (4996 data bytes plus 4 bytes of
packet overhead generated by the ICA datastream manager).

Both the virtual driver and the server-side application can query the
maximum packet size. See DriverOpen for an example of querying the
maximum packet size on the client side.

## Flow Control

ICA virtual channels provide support for downstream (server to client)
flow control, but there is currently no support for upstream flow
control. Data received by the server is queued until used.

Some transport protocols such as TCP/IP provide flow control, while
others such as IPX do not. If data flow control is needed, you might
need to design it into your virtual channel.

Choose one of three types of flow control for an ICA virtual channel:
None, Delay, or ACK. Each virtual channel can have its own flow control
method. The flow control method is specified by the virtual driver
during initialization.

### None

ICA does not control the flow of data. It is assumed the client can
process all data sent. You must implement any required flow control as
part of the virtual channel protocol. This method is the most difficult
to implement but provides the greatest flexibility. The Ping example
does not use flow con trol and does not require it.

### Delay

Delay flow control is a simple method of pacing the data sent from the
server. When the client virtual driver specifies delay flow control, it
also provides a delay time in milliseconds. The server waits for the
specified delay time between each packet of data it sends.

### ACK

ACK flow control provides what is referred to as a sliding window. With
ACK flow control, the client specifies its maximum buffer size (the
maximum amount of data it can handle at any one time). The server sends
up to that amount of data. The client virtual driver sends an ACK ICA
packet when it completes processing all or part of its buffer,
indicating how much data was processed. The server can then send more
data bytes up to the number of bytes acknowledged by the client.

This ACK is not transparent—the virtual driver must explicitly construct
the ACK packet and send it to the server. The server sends entire
packets; if the next packet to be sent is larger than the window, the
server blocks the send until the window is large enough to accommodate
the entire packet.

## Windows Monitoring API

These APIs allow creating solutions that synchronize the visual aspects
of an application that runs on a host (Citrix Virtual Apps and Desktops) with
corresponding visual elements that are running on Citrix Workspace app for Windows for
Windows. The APIs consist of two different parts: client-side and
host-side. The client- side component exposes previously unavailable
functionality to third parties. This includes getting information about
the ICA window on the client desktop (such as handle, dimensions,
panning and scaling), the corresponding client window to a given host
window, and setting up a callback function to be called when the ICA
window changes. The host component is part of the WinFrame API, and
allows for tracking window positions on the host through kernel mode
calls.

The APIs provide the following features:

*  Allow efficient tracking of windows on a host through the WinFrame API.
*  Provide methods for synchronizing with the client desktop display with the Virtual Channel SDK.
*  Provide an improved visual experience and support to third-party applications for better ICA integration.

### Getting started

Headers: wdapi.h

Libraries: wdica30.lib

### Architecture

The APIs provide two distinct components with their own architectures:
host-side and client- side. The host component is part of the WinFrame
API, and provides updates on tracked windows. You can then communicate
this data to Citrix Workspace app for Windows so as to synchronize window
positions. The client - side component in the Virtual Channel SDK then
allows third parties to synchronize with the ICA window. It provides
them with information about the ICA window's dimensions and handle, as
well as whether it is panning or scaling. As a whole, the APIs allow
third-party applications to better integrate with ICA and provide a
better visual experience.

The client side extends the current WdQueryInformation system in the
Virtual Channel SDK to expose functionality that was previously
unavailable to third parties. Users call the pre-existing VdCallWd
function to call the WinStation driver's QueryInformation function which
performs the requested task.

### Samples

#### Get ICA Window information

This sample shows how information about the ICA window is gathered. It
populates a structure with information about the current state of the
ICA window. This includes its dimensions, handle, view area dimensions
and offset (for example, panning), as well as its current mode (for
example, scaling, panning, seamless).

```
WDQUERYINFORMATION wdQueryInfo; UINT16 uiSize;
int rc;
WDICAWINDOWINFO infoParam;
wdQueryInfo.WdInformationClass = WdGetICAWindowInfo;
wdQueryInfo.pWdInformation = &infoParam;
wdQueryInfo.WdInformationLength = sizeof(infoParam);
uiSize = sizeof(wdQueryInfo);
rc = VdCallWd(g_pVd, WDxQUERYINFORMATION, &wdQueryInfo, &uiSize);

if(CLIENT_STATUS_SUCCESS == rc)
{
 // Successfully populated infoParam with ICA window
 // information
}
```

#### Get corresponding client window

This sample shows how to get the corresponding client window for a given server window.

```
WDQUERYINFORMATION wdQueryInfo; UINT16 uiSize;
int rc;
HWND window = 0x42; // example server window handle
wdQueryInfo.WdInformationClass = WdGetClientWindowFromServerWindow;
wdQueryInfo.pWdInformation = &window;
wdQueryInfo.WdInformationLength = sizeof(window);
uiSize = sizeof(wdQueryInfo);
rc = VdCallWd(g_pVd, WDxQUERYINFORMATION, &wdQueryInfo, &uiSize);

if(CLIENT_STATUS_SUCCESS == rc)
{
 // Success, pWdInformation now points to the
//corresponding client window hwnd.
}
```

#### Register ICA Window callback

This sample shows how to register a callback function that is called
when the ICA window changes. It registers a user defined callback
function named Foo. Afterward, whenever the ICA window changes, Foo is
called with the current ICA window mode passed in. More information
about the ICA window is then gathered using WdGetICAWindowInfo
information class, as demonstrated in the first sample.

```
WDQUERYINFORMATION wdQueryInfo; UINT16 uiSize;
int rc;
WDREGISTERWINDOWCALLBACKPARAMS callbackParams;
callbackParams.pfnCallback = &Foo; // Your callback function wdQueryInfo.WdInformationClass = WdRegisterWindowChangeCallback;
wdQueryInfo.pWdInformation = &callbackParams;
wdQueryInfo.WdInformationLength = sizeof(callbackParams);
uiSize = sizeof(wdQueryInfo);
rc = VdCallWd(g_pVd, WDxQUERYINFORMATION, &wdQueryInfo, &uiSize);
if(CLIENT_STATUS_SUCCESS == rc) {
// Callback successfully registered.
// Function Foo will be called whenever the ICA window
// mode, position, or size changes.
}
```

#### Unregister ICA Window callback

```
WDQUERYINFORMATION wdQueryInfo; UINT16 uiSize;
int rc;
wdQueryInfo.WdInformationClass = WdUnregisterWindowChangeCallback wdQueryInfo.pWdInformation = &callbackParams.Handle;
// Previously returned handle wdQueryInfo.WdInformationLength = sizeof(callbackParams.Handle);
uiSize = sizeof(wdQueryInfo);
rc = VdCallWd(g_pVd, WDxQUERYINFORMATION, &wdQueryInfo, &uiSize); if(CLIENT_STATUS_SUCCESS == rc)
{
// Callback successfully unregistered
}
```

#### Programming guide

The APIs as a whole provide better window control for applications that
coordinate windows between the host and client desktop. In general, the
host uses the WinFrame API component of the API to track windows of
interest. The host listens on an assigned mail slot for tracking updates
about its windows. These updates are then communicated to Citrix
Citrix Workspace app, where they are used to properly position corresponding
windows. Citrix Workspace app uses the client-side portion of the APIs in the
Virtual Channel SDK to synchronize its windows with the ICA window.
Citrix Workspace app can be notified when the ICA window changes, and thus
make any necessary changes to other third-party applications.

#### Programming reference

##### Structures

### WDQUERYINFORMATION

Pre-existing structure passed to the WinStation driver's QueryInformation method. Stores input as well as resulting output.

```
{
WDINFOCLASS WdInformationClass;
LPVOID pWdInformation;
USHORT WdInformationLength;
USHORT WdReturnLength;
} WDQUERYINFORMATION, * PWDQUERYINFORMATION;
```

*  WdInformationonClass: Set to the enum value corresponding to the API function you want to call.
*  pWdInformation: Necessary input parameters, if any, for this function call. If the call returns anything, it is stored here as well.
*  WdInformationLength: Set to the size of the input to which pWdInformation point.
*  WdReturnLength: Filled in upon return; the size of the return value to which pWdInformation points.

### WDICAWINDOWINFO

Struct passed as input when using the WdGetICAWindowInfo information
class. Upon successful return, this is populated with information about
the ICA window.

```
typedef struct _WDICAWINDOWINFO
{
HWND hwnd;
WDICAWINDOWMODE mode;
UINT32 xWinWidth, yWinHeight, xViewWidth, yViewHeight;
INT xViewOffset, yViewOffset;
} WDICAWINDOWINFO, * PWDICAWINDOWINFO;
```

*  hwnd: ICA window handle.
*  mode: Current mode of the ICA window (for example, scaling, panning,seamless).
*  xWinWidth: Width of the ICA window.
*  yWinHeight: Height of the ICA window.
*  xViewWidth: Width of the ICA window's view area.
*  yViewHeight: Height of the ICA window's view area.
*  xViewOffset: How much the view area is offset in the x dimension (horizontal panning).
*  yViewOffset: How much the view area is offset in the y dimension (vertical panning).

### WDREGISTERWINDOWCALLBACKPARAMS

Struct passed as input when using the WdRegisterWindowChangeCallback information class.

```
typedef struct _WDREGISTERWINDOWCALLBACKPARAMS
{
 PFNWD_WINDOWCHANGED pfnCallback;
 UINT32 Handle;
} WDREGISTERWINDOWCALLBACKPARAMS,*PWDREGISTERWINDOWCALLBACKPARAMS;
```

*  pfnCallback: The user defined function to be called when the ICA window changes (for example, its mode, dimensions, view). This function should have the following header, with the UINT parameter being the current mode of the ICA window (see WDICAWINDOWMODE): typedef VOID (cdecl * PFNWD_WINDOWCHANGED) (UINT32);

*  Handle: Upon successful return this handle is populated. It can later be used to identify the handle when unregistering the callback.

#### Unions

### WDICAWINDOWMODE

Union used to store the ICA window's current mode.

```
typedef union _WDICAWINDOWMODE
{
struct {
 UINT Reserved : 1;
 UINT Seamless : 1;
 UINT Panning : 1;
 UINT Scaling : 1;
} Flags;
 UINT Value;
} WDICAWINDOWMODE;
```

### Enumerations

The WDINFOCLASS enumeration has four values used by the Windows
Monitoring API:

*  WdGetICAWindowInfo
*  WdGetClientWindowFromServerWindow
*  WdRegisterWindowChangeCallback
*  WdUnregisterWindowChangeCallback

## Citrix Dynamic Virtual Channel Protocol

### Architecture

The primary purpose of the DVC protocol is to provide a generic
connection-based communication infrastructure over traditional Static
Virtual Channels (SVCs).

Dynamic Virtual Channels (or DVCs) are multiplexed over SVCs. In
general, one SVC is used per technology remoted over ICA. The DVC
protocol provides the ability to create and communicate between
logically connected dynamic virtual channel endpoints.

A Dynamic Virtual Channel is an end-to-end connection created between an
application running on the ICA host (first endpoint) and an application
running on the ICA client (second endpoint, referred to as DVC
listener). The end-to-end DVC connection is established and maintained
over an ICA connection.

Individual DVC instances are created and maintained by DVC managers.
There is a DVC manager running on the host (implemented as a device
driver and service) and another on the client (imple mented as a virtual
driver DLL). The host is responsible for creating dynamic virtual
channels and the client is responsible for creating and maintaining
connections to client-side DVC applications.

Once the DVC connection is established, both the host and the
client-side DVC applications can send data messages to each other. These
messages can be initiated by either side, and sending and receiving a
message is the same on either side.

The protocol allows for multiple static channels to be used for DVC. By
default, each DVC plug -in, representing a specific technology, runs on
a separate SVC. This allows the administrator to prioritize individual
DVC-remoted technologies by managing the priority of their respective
SVC. However, the DVC client can also be configured such that a SVC can
be shared by two or more DVC Plug -ins. This may be desirable in the
rare case when the number of DVC plug-ins is more than the available
SVCs. Currently a maximum of 64 SVCs are supported over ICA.

### How to write DVC component over ICA

Microsoft’s DVC is implemented over the Remote Desktop Protocol and the
Citrix DVC protocol is implemented over the ICA protocol. To write the
DVC component over ICA, Microsoft’s DVC API can be used.

The Microsoft DVC client-side APIs are found in: [http://msdn.microsoft.com/en-us/library/bb540853(VS.85).aspx](http://msdn.microsoft.com/en-us/library/bb540853\(/VS.85\)/.aspx)

The server-side APIs are found in: [http://msdn.microsoft.com/en-us/library/bb540857(VS.85).aspx](http://msdn.microsoft.com/en-us/library/bb540857\(/VS.85\)/.aspx)

### Citrix Dynamic Virtual Channel Setup

The following steps occur during the lifetime of a dynamic virtual
channel:

1.  The client DVC Manager enumerates all the registered DVC plug-ins
    and sends the list to the host, which consists of pairs of DVC
    Plug-in Friendly Name and corresponding SVC Name. The DVC plug- in
    Friendly Name could be a DLL name or any other friendly
    name available. The SVC name is either administrator-assigned or
    generated from the friendly name. The SVC name is always truncated
    to 7 characters plus a NULL terminator (8 total), which is the SVC
    name size used by the ICA protocol. The list is sent as part of
    the CAPABILITY_DYNAMIC_VIRTUAL_CHANNEL ICA capability exchanged
    during the initial ICA handshake at the WinStation Driver (WD)
    level. See the ICA3.0 protocol specification (ica30.doc) for
    details on the format of CAPABILITY_DYNAMIC_VIRTUAL_CHANNEL.

Remarks:

*  Normally, DVC plug-ins are registered according to the requirements defined by Microsoft. Citrix provides additional DVC plug-in registration for two purposes:
*  Enumeration of known 3rd party plug-ins that are not properly registered and cannot be enumerated otherwise.
*  Optional assignment by administrator of explicit SVC name per plug-in.
*  If the SVC name is generated from a friendly name, as opposed to administrator-assigned, then accidental collision with other SVCs is avoided as follows:
*  First the original name is truncated to 6 characters. Then a decimal digit is appended starting from 0 and up to 9 such that the new name is unique. If the name still is not unique, then step b is performed.
*  First the original name is truncated to 5 characters. Then two decimal digits are appended starting from 0 and up to 99 such that the new name is unique.

*  Name collision may occur with both SVCs used for DVC and other standard Citrix or 3rd party SVCs. The range from 0 to 99 used to create unique SVC names is sufficient, since currently ICA support only up to 64 SVCs.

*  The Client DVC Manager registers N number of SVCs with the WD in DriverOpen. In general, this will be 1 SVC per DVC Plug-in enumerated. However, plug-ins may share the same SVC name if:

*  The administrator has explicitly assigned the same SVC name for more than one DVC Plug in via the Citrix ICA Client DVC Registration.

*  There are no more SVCs available.
*  In the future the Client DVC Manager might assign and load a separate SVC per DVC listener (as opposed to plug-in). Currently, this is not possible because new listeners may become available at any point after loading of a plug-in and the current ICA architecture does not allow dynamic loading of SVCs at the client. All SVCs are loaded during the ICA handshake.

&#50;.  The host DVC Manager opens a SVC for each channel name received from
    the client via the CAPABILITY_DYNAMIC_VIRTUAL_CHANNEL
    ICA capability.

&#51;.  The host sends a list of supported DVC capabilities to the client
    and requests the list of client-supported capabilities. Currently,
    for optimization purposes, DVC capabilities are negotiated over
    one of the opened SVCs and they are assumed to apply to all SVCs.

&#52;. The client responds with a list of supported capabilities. The list
    is sent over the same SVC.

&#53;.  The host then commits the DVC capabilities to be used for the
    lifetime of the ICA connection. The list is sent over the
    same SVC.

&#55;  For each DVC Plug-in the client sends a list of all currently
    available listeners hosted by the DVC Plug-in. Each list is sent
    over the respective SVC:

*  Immediately after the DVC capabilities are committed by
        the host.

*  And at any point a new listener starts or an existing listener
        shuts down.

Remarks: Although it is theoretically possible for a listener to shut
down at any point, in practice once a listener starts, it does not shut
down until the whole DVC plug-in shuts down

&#56;  The host reads the list of listeners, caches any future updates from
    the client and keeps a mapping in a table.

&#57;.  When subsequently the Virtual Channel Open
    API (WTSVirtualChannelOpenEx) is called by a host application, the
    host looks up the listener name in its table, assigns a new
    Channel Number in the range from 0 to 64K and links it to the
    respective listener. The Channel Number is unique within
    a listener. A DVC Create request is then sent over the SVC
    associated with the listener. The client responds over the same
    SVC with success or failure to create the channel (DVC instance)
    on the specified listener. The client’s response is communicated
    to the host application.

&#49;&#48;.  If a DVC channel is successfully created, data messages can be
    exchanged between the application running on the ICA host and the
    DVC Listener running on the ICA client. Sending and receiving
    messages is symmetrical between the host and client, and either
    side can initiate sending a message.

&#49;&#49;. Eventually a DVC channel is closed by either the host or the client.
    A close is triggered when the host application closes the DVC
    instance handle but can also be triggered by a client listener or
    by the client or host DVC Managers, for example, upon error.

&#49;&#50;. All DVC packets are sent over SVC packets.

&#49;&#51;. When the ICA client exists, the client DVC Manager shuts down and
    unloads all the DVC Plug-ins, which in turn shut down
    their listeners.

### Naming static virtual channel

The Channel Name field provides a name for the static virtual channel
to use for a specific DVC plug-in. By default the static channel name
to use will be automatically generated using the module file name of
the DVC plug-in. To ensure that a unique name is generated, upon
collision one or two digits may be used at the end of the name to make
it unique while keeping the name length at a maximum of seven
characters. The channel name field is explained as follows:

Section: ChannelName

Feature: DVC

Attribute Name: INI_DVC_PLUGIN_&lt;DVC plugin name&gt;

Definition location: inc\icaini.h

Data Type: String
>
Access Type: Read
>
Unix Specific: No
>
Present in ADM: No
>
Values:
>
Static virtual channel name
>
INI Location:

  ---------------------- -------------------------------------------- --------- ---------
  INI File             Section                                              Value

  Module.ini           \[DVC_Plugin_&lt;DVC plugin name&gt;\
  Registry Location:
  Registry Key                                                      Value
  ---------------------- -------------------------------------------- --------- ---------

HKEY_LOCAL_MACHINE\SOFTW ARE\Citrix\ICA
Client\Engine\Configuration\Advanced\Modules\DVC_Plugin_&lt;DVC
plugin name&gt;
>
The static virtual channel name can be modified using the above
locations if the administrator wants to give the explicit name.
>

### Steps to write DVC component over ICA

This section explains how to write a DVC component over ICA using an
example. Citrix DVC protocol uses the existing interfaces provided by
Microsoft to develop DVC components over ICA. For more details on how to
write DVC Server components and DVC Client components refer to:

[*http://msdn.microsoft.com/en-us/library/bb540858(VS.85).aspx*](http://msdn.microsoft.com/en-us/library/bb540858(VS.85).aspx)

and

[*http://msdn.microsoft.com/en-us/library/bb540854(VS.85).aspx*](http://msdn.microsoft.com/en-us/library/bb540854(VS.85).aspx)

respectively.