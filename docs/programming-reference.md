# Programming Reference

For function summaries , see:

-  Server-Side Functions Overview
-  Client-Side Functions Overview

## DriverClose

The WinStation driver calls this function prior to unloading the
virtual driver, when the ICA connection is being terminated.

### Calling Convention

```
INT Driverclose
 ( PVD pVD,
 PDLLCLOSE pVdClose,
 PUINT16 puiSize);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVD | Pointer to a virtual driver control structure. |
| pVdClose | Pointer to a standard driver close information structure. |
| puiSize | Pointer to the size of the driver close information structure. This is an input parameter. |

### Return Values

If the function succeeds the return value is CLIENT_STATUS_SUCCESS.

If the function fails, the return value is the CLIENT_ERROR_\* value
corresponding to the error condition; see clterr.h (in src/inc/) for a
list of error values beginning with CLIENT_ERROR.

### Remarks

When DriverClose is called, all private driver data is freed. The
virtual driver does not need to deallocate the virtual channel or write
hooks.

The pVdClose structure currently contains one element – NotUsed. This
structure can be ignored.

## DriverGetLastError

This function is not used but is available for linking with the common
front end, VDAPI.

### Calling Convention

```
INT DriverGetLastError(
PVD pVD,
PVDLASSTERROR pVdLastError);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVD | Pointer to a virtual driver control structure. |
| pVdLast Error | Pointer to a structure that receives the last error information.|

### Return Value

The driver returns CLIENT_STATUS_SUCCESS.

### Remarks

This function currently has no practical significance for virtual
drivers; it is provided for compatibility with the loadable module
interface.

## DriverInfo

Gets information about the virtual driver, such as the version level of
the driver.

### Calling Convention

```
INT DriverInfo(
PVD pVD,
PDLLINFO pVdInfo,
PUINT16 puiSize);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVD | Pointer to a virtual driver control structure. |
| pVdInfo| Pointer to a standard driver information structure.|
| puiSize | Pointer to the size of the driver information structure. This is an output parameter. |

### Return Value

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails because the buffer pointed to by pVdInfo is too
small, it returns CLIENT_ERROR_BUFFER_TOO_SMALL. Normally, when a
CLIENT_ERROR_ result code is returned, the ICA session is
disconnected. CLIENT_ERROR_BUFFER_ TOO_SMALL is an exception and
does not result in the ICA session being disconnected. Instead, the
WinStation driver attempts to call DriverInfo again with the ByteCount
of pVdInfo returned by the failed call.

### Remarks

When the client starts, it calls this function to retrieve
module-specific information for transmission to the host. This
information is returned to the server side of the virtual channel by
WFVirtualChannelQuery.

The virtual driver must support this call by returning a structure in
the pVdInfo buffer. This structure can be a developer-defined virtual
channel-specific structure, but it must begin with a VD_C2H structure,
which in turn begins with a MODULE_C2H structure. All fields of the
VD_C2H structure must be filled in except for the ChannelMask field.
See ica-c2h.h (in src/inc/) for definitions of these structures.

The virtual driver must first check the size of the information buffer
given against the size that the virtual driver requires (the VD_C2H
structure). The size of the input buffer is given in pVdInfo
->ByteCount.

If the buffer is too small to store the information that the driver
needs to send, the correct size is filled into the ByteCount field and
the driver returns CLIENT_ERROR_BUFFER_TOO_SMALL.

If the buffer is large enough, the driver must fill it with a
module-defined structure. At a minimum, this structure must contain a
VD_C2H structure. The VD_C2H structure must be the first data in the
buffer; additional channel-specific data can follow. All relevant fields
of this structure are filled in by this function. The flow control
method is specified in the VDFLOW structure (an element of the VD_C2H
structure). The Ping example contains a flow control selection.

The WinStation driver calls this function twice at initialization, after
calling DriverOpen. The first call contains a NULL information buffer
and a buffer size of zero. The driver is expected to fill in
pVdInfo->ByteCount with the required buffer size and return
CLIENT_ERROR_BUFFER_TOO_SMALL. The WinStation driver allocates a
buffer of that size and retries the operation.

The data buffer pointed to by pVdinfo->pBuffer must not be changed by
the virtual driver. The WinStation driver stores byte swap information
in this buffer.

The parameter puiSize must be initialized to the size of the driver
information structure.

## DriverOpen

Initializes the virtual driver. The client engine calls this
user-written function once when the client is loaded.

### Calling Convention

```
INT DriverOpen(
PVD pVD, PVDOPEN pVdOpen)
PUINT16 puiSize);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVD | Pointer to the virtual driver control structure. This pointer is passed on every call to the virtual driver.
| pVdOpen | Pointer to the virtual driver Open structure.|
| puiSize | Pointer to the size of the virtual driver Open structure. This is an output parameter. |

### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails, it returns the CLIENT_ERROR_* value
corresponding to the error conditio n; see clterr.h (in src/inc/) for a
list of error values beginning with CLIENT_ERROR

### Remarks

The code fragments in this section are taken from the vdping example.

The DriverOpen function must:

1.  Allocate a virtual channel.Fill in a WDQUERYINFORMATION structure and call VdCallWd. The WinStation driver fills in the OpenVirtualChannel structure (including the channel number) and the data in pVd.

```
WDQUERYINFORMATION wdqi;
OPENVIRTUALCHANNEL OpenVirtualChannel;
wdqi.WdInformationClass = WdOpenVirtualChannel; wdqi.pWdInformation = &OpenVirtualChannel;
wdqi.WdInformationLength = sizeof(OPENVIRTUALCHANNEL); OpenVirtualChannel.pVCName = CTXPING_VIRTUAL_CHANNEL_NAME; rc = VdCallWd(pVd, WDxQUERYINFORMATION, &wdqi);
/* do error processing here */
```

After the call to VdCallWd, the channel number is assigned in the
OpenVirtualChannel structure's Channel element. Save the channel
number and set the channel mask to indicate which channel this driver
will handle.

For example:

```
g_usVirtualChannelNum = OpenVirtualChannel.Channel;
pVdOpen-&gt;ChannelMask = (1L &lt;&lt; OpenVirtualChannel-&gt;);
```

&#50;. Optionally specify a pointer to a private data structure.

If you want the virtual driver to allocate memory for state data, it can
have a pointer to this data returned on each call by placing the pointer
in the virtual driver structure, as follows:

```

pVd->pPrivate = pMyStructure;
```

&#51;.  Exchange entry point data with the WinStation driver.

The virtual driver must register a write hook with the client WinStation
driver. The write hook is the entry point of the virtual driver to be
called when data is received for this virtual channel. The WinStation
driver returns pointers to functions that the driver must use to fill in
output buffers and sends data to the WinStation driver for transmission
to the server.

```
WDSETINFORMATION wdsi; \]
VDWRITEHOOK vdwh;

// Fill in a write hook structure

vdwh.Type = g_usVirtualChannelNum;
vdwh.pVdData = pVd;
vdwh.pProc = (PVDWRITEPROCEDURE) ICADataArrival;

// Fill in a set information structure
wdsi.WdInformationClass = WdVirtualWriteHook;
wdsi.pWdInformation = &vdwh;
wdsi.WdInformationLength = sizeof(VDWRITEHOOK);
rc = VdCallWd( pVd, WDxSETINFORMATION, &wdsi);
/* do error processing here */
```

During the registration of the write hook, the WinStation driver passes
entry points for the output buffer virtual driver helper functions to
the virtual driver in the VDWRITEHOOK structure. The DriverOpen function
saves these in global variables so helper functions in the virtual
driver can use them. The WinStation driver also passes a pointer to the
WinStation driver data area, which the DriverOpen function also saves
(because it is the first argument to the virtual driver helper
functions).

```
// Record pointers to functions used
// for sending data to the host.
pWd = vdwh.pWdData;
pOutBufReserve = vdwh.pOutBufReserveProc;
pOutBufAppend = vdwh.pOutBufAppenProc;
pOutBufWrite = vdwh.pOutBufWriteProc;
pAppendVdHeader = vdwh.pAppendVdHeaderProc;
```

&#52;. Determine the version of the WinStation driver.

New virtual drivers should determine whether the WinStation driver
supports the new SendData API and “no polling” mode. Use the WdVirtualWriteHookEx information class to retrieve this information:

```
// Do extra initialization to determine if
// we are talking to an HPC client.
wdsi.WdInformationClass = WdVirtualWriteHookEx;
wdsi.pWdInformation = &vdwhex;
wdsi.WdInformationLength = sizeof(VDWRITEHOOKEX);
vdwhex.usVersion = HPC_VD_API_VERSION_LEGACY;

//Set version

// to 0; older clients will do nothing
rc = VdCallWd(pVd, WDxQUERYINFORMATION, &wdsi, &uiSize);
if (CLIENT_STATUS_SUCCESS != rc)
{
 return(rc);
}
g_fIsHpc = (HPC_VD_API_VERSION_LEGACY != vdwhex.usVersion);

// If version returned, this is HPC or later
g_pSendData = vdwhex.pSendDataProc;  // save HPC SendData
 // API address
```

The usVersion that is returned may be one of the following values:

```
typedef enum _HPC_VD_API_VERSION
{
 HPC_VD_API_VERSION_LEGACY = 0,  // legacy VDs
 HPC_VD_API_VERSION_V1 =   1, // VcSDK API version 1
} HPC VD API VERSION;
```

If the usVersion returned is HPC_VD_API_VERSION_LEGACY, the engine
is an earlier engine. Any other value indicates the newer engine. The
actual version returned indicates the version of the API supported.
Currently the only other value that will be returned is
HPC_VD_API_VERSION_V1. The g_fIsHpc flag should be set to indicate
that the newer API is available.

The WdVirtualWriteHookEx call also returns a pointer (g_pSendData).
This is a pointer to the SendData function. Save this value for later
use.

&#53;. Set the API options in the WinStation driver.

If this virtual driver is loaded by the HPC WinStation driver, set the
API options this driver will use:

```
if(g_fIsHpc)
{
 WDSET_HPC_PROPERITES hpcProperties;
 hpcProperties.usVersion = HPC_VD_API_VERSION_V1;
 hpcProperties.pWdData = g_pWd;
 hpcProperties.ulVdOptions = HPC_VD_OPTIONS_NO_POLLING;
 wdsi.WdInformationClass = WdHpcProperties;
 wdsi.pWdInformation = &hpcProperties;
 wdsi.WdInformationLength = sizeof(WDSET_HPC_PROPERITES);
 rc = VdCallWd(pVd, WDxSETINFORMATION, &wdsi, &uiSize);
 if(CLIENT_STATUS_SUCCESS != rc)
 {
 eturn(rc);
 }
}
```

The usVersion field is set to inform the engine of the version of the VD
API that this driver will use. This allows the engine to maintain the
compatibility of the VD API for this driver at this level, even if the
engine API changes in the future.

The pWdData pointer must point to the same data that was pointed to by
the pWdData field returned by the WdVirtualWriteHook VdCallWd call
earlier in DriverOpen.

The ulVdOptions field is a bitwise OR of any of the following bit
definitions:

```
typedef enum _HPC_VD_OPTIONS
{
 HPC_VD_OPTIONS_NO_POLLING =0x0001,  // Flag indicating that
 //channels on this VD do not
 // require send data polling
 HPC_VD_OPTIONS_NO_COMPRESSION =0x0002  // Flag indicating
 // that channels on this VD
 // send data that does not
 // need reducer compression
} HPC_VD_OPTIONS;
```

&#54;.  Allocate all memory needed by the driver and do any initialization.
    You can obtain the maximum ICA buffer size from the
    MaximumWriteSize element in the VDWRITEHOOK structure that
    is returned.

!!!tip "Note"
 vdwh.MaximumWriteSize is one byte greater than the actual
maximum that you can use because it also includes the channel number.

```
g_usMaxDataSize = vdwh.MaxiumWriteSize - 1;
if(NULL == (pMyData = malloc( g_usMaxDataSize )))
{
 return(CLIENT_ERROR_NO_MEMORY);
}
```

&#55;  Return the size of the VDOPEN structure in `puiSize`. This is used
    by the client engine to determine the version of the virtual
    channel driver.

## DriverPoll

Allows the virtual driver to check timers and other state information,
send queued data to the server, and perform any other required
processing. This function may be called on a regular basis by the main
clie nt poll loop.

### Calling Convention

```
INT DriverPoll(
PVD pVD,
PVOID pVdPoll,
PUINT16 puiSize);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVD | Pointer to a virtual driver control structure.|
| pVdPoll | Pointer to one of the driver poll information structures (DLLPOLL). |
| puiSize | Pointer to the size of the driver poll information structure. This is an output parameter. |

### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the driver has no data on this polling pass, it returns
CLIENT_STATUS_NO_DATA.

If all virtual channels return CLIENT_STATUS_NO_DATA, the WinStation
driver may slow down the polling process.

If the sending of data via either the QueueVirtualWrite or the SendData
function is blocked (CLIENT_ERROR_NO_OUTBUF), DriverPoll should
return CLIENT_STATUS_ERROR_RETRY so the WinStation driver does not
slow polling. The virtual driver should then try again the next time it
is polled.

If the virtual driver cannot allocate an output buffer, it returns
CLIENT_STATUS_ERROR_RETRY so the WinStation driver does not slow
polling. The virtual driver then attempts to get an output buffer the
next time it is polled.

If polling has been disabled via the HPC_VD_OPTIONS_NO_POLLING
option, DriverPoll will be called at least once, and then only when the
virtual driver has asked to be polled, or when it has asked to be
notified when a send operation can be retried. The return values have
the same setting as in the polling case above. Return
CLIENT_STATUS_SUCCESS if all data has been sent successfully. Return
CLIENT_STATUS_NO_DATA if there is no data available to send. Return
CLIENT_STATUS_ERROR_RETRY if the send operation was blocked and the
virtual driver has more data to send.

Return values that begin with CLIENT_ERROR_\* are fatal errors; the
ICA session will be disconnected.

### Remarks

A virtual driver is not allowed to block while waiting for a desired
result (such as the availability of an output buffer).

The Ping example includes examples of processing types that can occur in
Driver Poll.

## DriverQueryInformation

Gets run-time information from the virtual driver.

### Calling Convention

```
INT DriverQueryInformation(
PVD pVD,
PVDQUERYINFORMATION pVdQueryInformation,
PUINT16 puiSize);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVD | Pointer to a virtual driver control structure.|
| pVdQuery Information | Pointer to a structure that specifies the information to query and the results buffer.|
| puiSize | Pointer to the size of the query information and resolves structure. This is an output parameter.|

### Return Value

The function returns CLIENT_STATUS_SUCCESS.

### Remarks

This function currently has no practical significance for virtual
drivers; it is provided for compatibility with the loadable module
interface. There are no general purpose query functions at this time
other than LastError. The LastError query is accomplished through the
DriverGetLastError function.

## DriverSetInformation

Sets run-time information in the virtual driver.

### Calling Convention

```
INT DriverSetInformation(
PVD pVD,
PVDSETINFORMATION pVdSetInformation,
PUINT16 puiSize);
```

### Parameters

| Parameter        | Description |
|:------------------:|:-------------:|
| pVD  | Pointer to a virtual driver control structure.|
| pVdSet Information | Pointer to a structure that specifies the information class, a pointer to any additional data, and the size in bytes of the additional data (if any).|
| puiSize | Pointer to the size of the information structure. This is an input parameter. |

### Return Value

The function returns CLIENT_STATUS_SUCCESS.

### Remarks

This function can receive two information classes:

-  VdDisableModule: When the connection is being closed.
-  VdFlush: When WFPurgeInput or WFPurgeOutput is called by the server-side virtual channel application. The VdSetInformation structure contains a pointer to a VDFLUSH structure that specifies which purge function was called.

## SendData

Sends a virtual channel packet to the server, with a notification
option.

### Calling Convention

```
INT WFCAPI SendData(DWORD pWd, USHORT usChannel,
 LPBYTE pData,USHORT usLen,
 LPVOID pUserData, UINT32 uiFlags);
```

### Parameters

| Parameters | Description |
|------------|-------------|
| pWd | Pointer to a WinStation driver control structure. |
| us Channel | The virtual channel number |
| pData |Data passed as an argument to the callback |
| callback.usLen | Lenght in bytes of the data in the data buffer |
|us User Data | Length in b ytes of the data in the data buffer |
| uiFlags | Flags to control the operation of the SendData function. This value consists of a number of flags bitwise OR'ed together. Each of the flags controls some aspect of the SendData interface. |

Currently there is only one flag defined. See the SENDDATA_\* enum:

-  SENDDATA_NOTIFY: If this flag is set,and when the SendData return code is CLIENT_ERROR_NO_OUTBUF indicating that the engine had no buffers to accommodate the outbound packet, the engine will notify the virtual driver later when it can retry the send operation. The notification occurs via the DriverPoll method.

### Return Value

The SendData function will return one of the following values:

-  CLIENT_STATUS_SUCCESS:
    -  The data was copied into virtual write buffers.
    -  The user's buffer is free.
    -  No callback will occur, even if the SENDDATA_NOTIFY flag is set.
    -  The next SendData call can be issued immediately.
-  CLIENT_ERROR_NO_OUTBUF:
    -  The virtual write could not be scheduled (out of
    VirtualWrite buffers). If a notification was requested, DriverPoll
    will be driven with the notification at some later time when the
    virtual driver should retry sending.
    -  If no notification was requested, the virtual driver should return
    from DriverPoll and wait for the next poll before retrying
    the send. This assumes that the virtual driver had selected the
    polled mode of operation.

-  CLIENT_ERROR_BUFFER_STILL_BUSY: If the user has called SendData requesting a notification, and the return code was CLIENT_ERROR_NO_OUTBUF, the user must not issue another SendData call until the notification has occurred. If another call is issued before the notification occurs, the     `CLIENT_ERROR_BUFFER_STILL_BUSY` return code will result.

-  CLIENT_ERROR_*: Any other error should cause the virtual driver and the session to close.

> Note
>
> If the user has specified HPC_VD_OPTIONS_NO_POLLING in the HPC
channel options, then the virtual driver must assume that its DriverPoll
function will not be called again after receiving one of these errors.

### Remarks

This function is used to send channel protocol to the server. The engine
either accepts all the data, or refuses it all, in which case the
channel will need to retry later (normally inside DriverPoll).

The address for this function is obtained from the VDWRITEHOOKEX
structure after hook registration in pSendDataProc. The VDWRITEHOOK
structure provides pWd.

## ICADataArrival

The WinStation driver calls this function when data is received on a
virtual channel being monitored by the driver. The address of this
function is passed to the WinStation driver during DriverOpen.

### Calling Convention

```
VOID wfcapi ICADataArrival(
 PVD pVD,
 USHORT uchan,
 LPBYTE pBuf,
 USHORT Length);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVD | Pointer to a virtual driver control structure. |
| uChan | Virtual channel number. |
| pBuf | Pointer to the data buffer containing the virtual channel data as sent by the server-side application. |
| Length | Length in bytes of the data in the buffer. |

### Return Value

No value is returned from this function.

### Remarks

This function name is a placeholder for a user-defined function; the
actual function does not have to be called ICADataArrival, although it
does have to match the function signature (parameters and return type).
The address of this function is given to the WinStation driver during
DriverOpen. Although ICA prefixes packet control data to the virtual
channel data, this prefix is removed before this function is called.

After the virtual driver returns from this function, the WinStation
driver considers the data de livered. The virtual driver must save
whatever information it needs from this packet if later processing is
required.

Do not allow this function to block. Use your own thread or the
DriverPoll function (with polling enabled) for any required deferred
processing.

The virtual driver can send data to the server on receipt of this data
from within the ICADataArrival function, but be aware that the send
operation may return an immediate error when buffers are not available
to accommodate the send operation. The virtual driver may not block in this
function waiting for the sending operation to complete.

If the virtual driver is handling multiple virtual channels, use the
uChan parameter to determine the channel over which this data is to be
sent. See DriverOpen for more information.

## miGetPrivateProfileBool

Gets a Boolean value from a section of the Configuration Storage.

### Calling Convention

```
INT miGetPrivateProfileBool(
 PCHAR lpszSection,
 PCHAR lpszEntry,
 BOOL bDefault);
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| lpsz Section | Name of section to query. |
| lpsz Entry | Name of entry to query. |
| bDefault | Default value to use. |

### Return Values

If the requested entry is found, the entry value is returned; otherwise,
`bDefault` is returned.

### Remarks

A Boolean value of TRUE can be represented by on, yes, or true in the
configuration files. All other strings are interpreted as FALSE.

## miGetPrivateProfileInt

Gets an integer from a section of the Configuration Storage.

### Calling Convention

```
INT miGetPrivateProfileInt(
 PCHAR lpszSection,
 PCHAR lpszEntry,
 INT iDefault);


### Parameters
| Parameter | Description |
|-----------|-------------|
| lpsz Section | Name of section to query. |
| lpsz Entry | Name of entry to query. |
| iDefault | Default value to use. |


### Return Values

If the requested entry is found, the entry value is returned; otherwise,
iDefault is returned.

## miGetPrivateProfileLong

Gets a long value from a section of the configuration files.

### Calling Convention

```

INT miGetPrivateProfileLong(
 PCHAR lpszSection,
 PCHAR lpszEntry,
 LONG lDefault);

```

### Parameters

| Parameter | Description |
|-----------|-------------|
| lpsz Section | Name of section to query. |
| lpsz Entry | Name of entry to query. |
| iDefault | Default value to use. |

### Return Values

If the requested entry is found, the entry value is returned; otherwise,
lDefault is returned.


## miGetPrivateProfileString

Gets a string from a section of the configuration files.

### Calling Convention

```

INT miGetPrivateProfileString(
 PCHAR lpszSection,
 PCHAR lpszEntry,
 PCHAR lpszDefault,
 PCHAR lpszReturnBuffer, INT cbSize);

```

### Parameters

| Parameter | Description |
|-----------|-------------|
| lpsz Section | Name of section to query. |
| lpsz Entry | Name of entry to query. |
| lpsz Default | Default value to use. |
| lpsz Return Buffer | Pointer to a buffer to hold results. |
| cb Size | Size of lpszReturnBuffer in bytes. |


### Return Values

This function returns the string length of the value returned in
lpszReturnBuffer (not including the trailing NULL).

If the requested entry is found and the size of the entry string is less
than or equal to cbSize, the entry value is copied to lpszReturnBuffer;
otherwise, iDefault is copied to lpszReturnBuffer.

### Remarks

lpszDefault must fit in lpszReturnBuffer. The caller is responsible for
allocating and deallocating lpszReturnBuffer.

lpszReturnBuffer must be large enough to hold the maximum length entry
string, plus a NULL termination character. If an entry string does not
fit in lpszReturnBuffer, the lpszDefault value is used.


## QueueVirtualWrite

A QueueVirtualWrite is an improved scatter gather interface. It queues a
virtual write and stimulates packet output if required allowing data to
be sent without having to wait for the poll.

### Calling Convention

```

int WFCAPI
QueueVirtualWrite (
PWD pWd,
SHORT Channel,
LPMEMORY_SECTION pMemorySections,
USHORT NrOfSections,
USHORT Flag);

```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pWd | Pointer to a WinStation driver control structure. |
| Channel | The virtual channel number |
| pMemory Sections | Pointer to an array of one or more elements of the structure MEMORY_SECTION (see below) containing the pure body of the protocol, i.e. excluding the virtual write header defining the channel and the length. This will get constructed by the QueueVirtualWrite function itself. Normally the protocol body will already be in a single contiguous block of memory. But if not, then multiple sections can be defined, and the destination function will copy all the different pieces into the appropriate internal WD queue.
| Number Of Sections | The number of memory sections, normally 1. |
| Flush Control | Indicates whether a ‘flush to wire’ operation should be triggered after the data has been successfully queued. If this value is FLUSH_IMMEDIATELY, then if the line conditions permit, the WD will attempt to send this new virtual write immediately over the wire, after also flushing any earlier queued data for the same or higher priority. If the data is not time critical (within the span of about 50ms), it may be better not to force a flush at this point, so that the data (if small) may go over the wire together with other data, so making better use of the wire bandwidth. The value to use if an immediate flush is not required is !FLUSH_IMMEDIATELY. |

### Memory section

This structure has the definition:

```

typedef struct _MEMORY_SECTION
{
 UINT lenght; //Length of data
 LPBYTE pSection;  //Address of data
} MEMORY SECTION, far * LPMEMORY SECTION;

```

### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS. If it
cannot currently accept the data, it returns CLIENT_ERROR_NO_OUTBUF.
If being called from DriverPoll, then the return value for the
DriverPoll should normally be set to CLIENT_STATUS_ERROR_RETRY in
this case. If the function fails, it returns an error code associated
with the failure; use GetLastError to get the extended error
information.

### Remarks

This function is used to send channel protocol to the server. The engine
either accepts all the data, or refuses it all, in which case the
channel will need to retry later (normally inside DriverPoll).

The address for this function is obtained from the VDWRITEHOOK structure
after hook Registration in pQueueVirtualWriteProc. The VDWRITEHOOK
structure also provides pWd. Each successful call will ultimately result
in a single block of protocol, with length = total length of all memory
sections, being delivered to the server-side channel.

## VdCallWd

Calls the client WinStation driver to query and set information about
the virtual channel. This is the main method for the virtual driver to
access the WinStation driver. For general-purpose virtual channel
drivers, this sets the virtual write hook.

### Calling Convention

```

INT VdCallWd (
PVD pVd,
USHORT ProcIndex,
PVOID pParam);

```

### Parameters

| Parameter | Description |
|-----------|-------------|
| pVd | Pointer to a virtual driver control structure. |
| Proc Index | Index of the WinStation driver routine to call. For virtual drivers, this can be either `WDxQUERYINFORMATION` or `WDxSETINFORMATION`.|
| pParam | Pointer to a parameter structure. |


### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails, it returns an error code associated with the
failure; use DriverGetLastError to get the extended error information.

### Remarks

This function is a general purpose mechanism to call routines in the
WinStation driver. The only valid uses of this function for a virtual
driver are to:.

* To allocate the virtual channel using WDxQUERYINFORMATION
* To exchange function pointers with the
WinStation driver during DriverOpen using WDxSETINFORMATION

For more information, see DriverOpen or the Ping example.

On successful return, the VDWRITEHOOK structure contains pointers to the
output buffer virtual driver helper functions, and a pointer to the
WinStation driver control block (which is needed for buffer calls).

## WFVirtualChannelClose

Closes an open virtual channel handle.

### Calling Convention

```

BOOL WINAPI WFVirtualChannelClose(IN HANDLE hChannelHandle);

```

### Parameter

| Parameter | Description |
|-----------|-------------|
| hChannel Handle | Handle to a previously opened virtual channel. Use WFVirtualChannelOpen to obtain a handle for a specific channel. |


### Return Values

If the function succeeds, the return value is TRUE. If the function
fails, the return value is FALSE; call GetLastError to get extended
error information.

### Remarks

When this function is called, any data waiting to be sent to the client
is discarded. Call this function when the server-side virtual channel
application is finished.

The client- side virtual driver is not closed until the ICA session is
closed. If the virtual driver sends data to the server after the
server-side application closes, the data is queued on the server and
eventually discarded.

To prevent the virtual driver from sending data after the server-side
application closes, you might need to incorporate a packet type into
your virtual channel protocol to notify the virtual driver that the
server-side application is closing.

## WFVirtualChannelOpen

Opens a handle to a specific virtual channel.

### Calling Convention

```

HANDLE WINAPI WFVirtualChannelOpen(IN HANDLE hServer,
 IN DWORD SessionId,
 IN LPSTR pVirtualName // ANSI name
 );

```

### Parameters

| Parameter | Description |
|-----------|-------------|
| hServer | Handle to a server running Citrix Virtual Apps. To specify the current server, use the constant WF_CURRENT_SERVER_HANDLE. Use WFOpenServer to obtain a handle for a specific server. For more information about the WFOpenServer function, see the WFAPI SDK documentation. |
| Session Id | Server session ID. Use the constant WF_CURRENT_SESSION to specify the current session. To obtain the session ID of a specific session, use WFEnumerateSessions. For more information about session IDs and WFEnumerateSessions, see the WFAPI SDK documentation. |
| pVirtual Name | Pointer to the virtual channel name. This is an ASCII string (even when Unicode is defined) of no more than seven characters.


### Return Values

If the function succeeds, it returns a handle to the specified virtual
channel. If the function fails, it returns NULL; call GetLastError for
extended error information.

### Remarks

The WinStation driver opens the channel by name, assigns a channel
number, and returns a handle. The server -side virtual channel
application uses this handle to read and write data to the virtual
channel.

## WFVirtualChannelPurgeInput

Purges all queued input data sent from the client to the server on a
specific virtual channel.

### Calling Convention

```

BOOL WINAPI WFVirtualChannelPurgeInput(IN HANDLE hChannelHandle);

```

### Parameter

| Parameter | Description |
|-----------|-------------|
| hChannel Handle | Handle to a previously opened virtual channel. To obtain a handle for a specific channel, use WFVirtualChannelOpen. |

### Return Values

If the function succeeds, the return value is TRUE. If the function
fails, the return value is FALSE; call GetLastError to get extended
error information.

### Remarks

Output buffers and queued data received from the client are discarded.

This function sends a message to the client WinStation driver, which
then calls the client virtual driver’s

DriverSetInformation function with the VdFlush information class. For
most virtual channels, this function is not necessary and you can use
the Ping example function without modification.

## WFVirtualChannelPurgeOutput

Purges all queued output data sent from the server to the client on a
specific virtual channel.

Example of use: in an audio application in which the user starts playing
a different audio file, use this function to discard the audio that was
queued to be sent to the client from the first file played.

### Calling Convention

```

BOOL WINAPI WFVirtualChannelPurgeOutput(IN HANDLE hChannelHandle);

```

### Parameter

| Parameter | Description |
|-----------|-------------|
| hChannel Handle | Handle to a previously opened virtual channel. To obtain a handle for a specific channel, use WFVirtualChannelOpen. |

### Return Values

If the function succeeds, the return value is TRUE. If the function
fails, the return value is FALSE. Call GetLastError to get extended
error information.

### Remarks

Output buffers and data queued to be sent to the client are discarded.

This function sends a message to the client WinStation driver, which
then calls th e client virtual driver’s DriverSetInformation function
with the VdFlush information class. For most virtual channels, this
function is not necessary and you can use the Ping example function
without modification.

## WFVirtualChannelQuery

Returns data related to a virtual channel. This information is obtained
when the ICA connection is initiated and the WinStation driver calls the
DriverInfo function.

### Calling Convention

```

BOOL WINAPI WFVirtualChannelQuery(IN HANDLE hChannelHandle,
   IN WF_VIRTUAL_CLASS VirtualClass,
   OUT PVOID \*ppBuffer,
    OUT DWORD \*pBytesReturned
   );

```

### Parameters

| Parameter | Description |
|-----------|-------------|
| hChannel Handle | Handle to a previously opened virtual channel. To obtain a handle for a specific channel, use WFVirtualChannelOpen. |
| Virtual Class | Type of information to request. Currently, the only defined value is WFVirtualClientData , which returns virtual channel client module data. |
| ppBuffer | Pointer to the address of a variable to receive the data. The buffer is allocated within this fun ction and is deallocated by using WFFreeMemory. |
| pBytes Returned | Pointer to a DWORD that is updated with the length of the data returned in the allocated buffer (upon successful return). |

### Return Values

If the function succeeds, the return value is TRUE. If the function
fails, the return value is F ALSE; Call GetLastError to get extended
error information.

### Remarks

ppBuffer begins with the structure VD_C2H, which begins with the
structure MODULE_C2H. These two structures are defined in Ica-c2h.h (in
src\inc\). See the Ping example for more information.

## WFVirtualChannelRead

Reads data from a virtual channel.

### Calling Convention

```

BOOL WINAPI WFVirtualChannelRead(IN HANDLE hChannelHandle,
  IN ULONG TimeOut,
 OUT PCHAR pBuffer,
  IN ULONG BufferSize,
  OUT PULONG pBytesRead
  );

```
### Parameters

| Parameter | Description |
|-----------|-------------|
| hChannel Handle | Handle to a previously opened virtual channel. To obtain a handle for a specific channel, use WFVirtualChannelOpen. |
| TimeOut | Length of time to wait for data (in milliseconds). If this value is zero, the function returns immediately whether or not data is available. If this value is -1, the function continues waiting until there is data to read. |
| pBuffer | Buffer to receive the data read from the virtual channel. |
| Buffer Size | Size in bytes of the buffer needed. |
| pBytes Read | Pointer to a variable that receives the number of bytes read. |

### Return Values

If the function succeeds, the return value is TRUE. If the function
fails, the return value is FALSE; call GetLastError to get extended
error information.

### Remarks

The developer determines the BufferSize, which can be any length up to
the maximum size supported by the ICA connection. This size is
independent of size restrictions on the lower-layer transport.

* If the server is running Citrix Virtual Apps, the
maximum packet size is 5000 bytes (4996 bytes of data plus the 4-byte
packet overhead generated by the ICA datastream manager).

If more data is received than the buffer can hold, the entire packet is
discarded.

If no data is received, pBuffer and pBytesRead are unmodified. The
function fails if the read times out.

The server-side virtual channel application is not notified when data is
received. Instead, the data is queued until the application uses
WFVirtualChannelRead to retrieve it.

## WFVirtualChannelWrite

Writes data to a virtual channel.

### Calling Convention

```

BOOL WINAPI WFVirtualChannelWrite (IN HANDLE hChannelHandle,
 IN PCHAR pBuffer,
 IN ULONG Length,
  OUT PULONG pBytesWritten
  );

```

### Parameters

| Parameter | Description |
|-----------|-------------|
| hChannel Handle | Handle to a previously opened virtual channel. To obtain a handle for a specific channel, use WFVirtualChannelOpen. |
| pBuffer | Buffer containing data to write to the virtual channel. This must be four bytes larger than the largest buffer written by the client. |
| Length | Size in bytes of the buffer needed. This must be four bytes larger than the data written by the application. |
| pBytes Written | Pointer to a ULONG variable that receives the number of bytes written. |

### Return Values

If the data is sent, the return value is TRUE.

If the data is not sent, the return value is FALSE; call GetLastError to
get extended error information.

### Remarks

The developer determines the Length, which can be any length up to the
maximum size supported by the ICA connection. This size is independent
of size restrictions on the lower-layer transport.

* If the server is running XenApp, the
maximum packet size is 5000 bytes (4996 bytes of data plus the 4-byte
packet overhead generated by the ICA datastream manager).

The WinStation driver calls the client virtual driver's ICADataArrival
function.
