# MacOSX-Network-Sockets-Filter

##License

The license model is a BSD Open Source License. This is a non-viral license, only asking that if you use it, you acknowledge the authors, in this case Slava Imameev.

Some NKE's project code commentary and ideas have been borrowed from Apple's tcplognke example. The code for tcplognke can be found here https://github.com/gdbinit/tcplognke .

##Directories structure

The project contains a NKE(Network Kernel Extension) module and a user mode client to communicate with the NKE.  
NKE directory contains a project for the kernel extension.  
NkeClient directory contains a project for a usermode client that responses to the NKE immediately without data inspection. The user client prints received events to console output.  

##Design

This is a MacOS network sockets filter, aka Network Kernel Extension(NKE). You can read more about NKE architecture here `https://developer.apple.com/library/content/documentation/Darwin/Conceptual/NKEConceptual/socket_nke/socket_nke.html`.

The filter registers IPv4 and IPv6 callbacks

```
/* Dispatch vector for IPv4 socket functions */
struct sflt_filter NkeSocketFilter::SfltIPv4 = {
	NKE_SOCK_FLT_HANDLE_IP4,/* sflt_handle - use a registered creator type - <http://developer.apple.com/datatype/> */
	SFLT_GLOBAL,			/* sf_flags */
	"NKEriverNKE",		    /* sf_name - cannot be nil else param err results */
	FltUnregisteredIPv4,	/* sf_unregistered_func */
	FltAttachIPv4,          /* sf_attach_func - cannot be nil else param err results */			
	FltDetachIPv4,			/* sf_detach_func - cannot be nil else param err results */
	FltNotify,              /* sf_notify_func */
	NULL,					/* sf_getpeername_func */
	NULL,					/* sf_getsockname_func */
	FltDataIn,              /* sf_data_in_func */
	FltDataOut,             /* sf_data_out_func */
	FltConnectIn,           /* sf_connect_in_func */
	FltConnectOut,          /* sf_connect_out_func */
	FltBind,				/* sf_bind_func */
	FltSetoption,           /* sf_setoption_func */
	FltGetoption,           /* sf_getoption_func */
	FltListen,              /* sf_listen_func */
	NULL					/* sf_ioctl_func */
};

/* Dispatch vector for IPv6 socket functions */
struct sflt_filter NkeSocketFilter::SFltIPv6 = {
	NKE_SOCK_FLT_HANDLE_IP6,/* sflt_handle - use a registered creator type - <http://developer.apple.com/datatype/> */
	SFLT_GLOBAL,			/* sf_flags */
	"NKEriverNKE",		    /* sf_name - cannot be nil else param err results */
	FltUnregisteredIPv6,	/* sf_unregistered_func */
	FltAttachIPv6,          /* sf_attach_func - cannot be nil else param err results */			
	FltDetachIPv6,			/* sf_detach_func - cannot be nil else param err results */
	FltNotify,              /* sf_notify_func */
	NULL,					/* sf_getpeername_func */
	NULL,					/* sf_getsockname_func */
	FltDataIn,              /* sf_data_in_func */
	FltDataOut,             /* sf_data_out_func */
	FltConnectIn,           /* sf_connect_in_func */
	FltConnectOut,          /* sf_connect_out_func */
	FltBind,				/* sf_bind_func */
	FltSetoption,           /* sf_setoption_func */
	FltGetoption,           /* sf_getoption_func */
	FltListen,              /* sf_listen_func */
	NULL					/* sf_ioctl_func */
};
```

Callbacks send event notifications to a user client. For input and output data (NkeSocketFilter::FltDataIn and NkeSocketFilter::FltDataOut callbacks respectively) the filter waits for a response from a usermode client. A user client processes sent or received data and then reply to the filter by filling a response structure and calling

```
	kr = IOConnectCallStructMethod( connection,
									kt_NkeUserClientSocketFilterResponse,
									(const void*)&response,
									sizeof(response),
									&response,
									&notUsed );
```

The filter processes a response from a client in NkeSocketFilter::processServiceResponse

```
    thread #3: tid = 0x162f, 0xffffff7fa3560710 NetworkKernelExtension`NkeSocketFilter::processServiceResponse(this=0xffffff8034838800, response=0xffffff802f2d727c) + 16 at NkeSocketFilter.cpp:1293, name = '0xffffff8031940a80', queue = '0x0', stop reason = breakpoint 1.1
    frame #0: 0xffffff7fa3560710 NetworkKernelExtension`NkeSocketFilter::processServiceResponse(this=0xffffff8034838800, response=0xffffff802f2d727c) + 16 at NkeSocketFilter.cpp:1293
    frame #1: 0xffffff7fa355d0e3 NetworkKernelExtension`NkeIOUserClient::processServiceSocketFilterResponse(this=0xffffff802c24b600, vInBuffer=0xffffff802f2d727c, vOutBuffer=0xffffff8034891600, vInSize=0x00000000000005c8, vOutSizeP=0xffffff80ba83bb88, (null)=0x0000000000000000, (null)=0x0000000000000000) + 163 at NkeIOUserClient.cpp:719
    frame #2: 0xffffff8021301c92 kernel`shim_io_connect_method_structureI_structureO(method=<unavailable>, object=<unavailable>, input=<unavailable>, inputCount=<unavailable>, output=<unavailable>, outputCount=<unavailable>) + 290 at IOUserClient.cpp:4341
    frame #3: 0xffffff8021302859 kernel`IOUserClient::externalMethod(this=<unavailable>, selector=<unavailable>, args=0xffffff80ba83bbe0, dispatch=<unavailable>, target=<unavailable>, reference=<unavailable>) + 841 at IOUserClient.cpp:4899
    frame #4: 0xffffff8021300003 kernel`is_io_connect_method(connection=0xffffff802c24b600, selector=2, scalar_input=<unavailable>, scalar_inputCnt=<unavailable>, inband_input=<unavailable>, inband_inputCnt=<unavailable>, ool_input=<unavailable>, ool_input_size=<unavailable>, inband_output=<unavailable>, inband_outputCnt=<unavailable>, scalar_output=<unavailable>, scalar_outputCnt=<unavailable>, ool_output=<unavailable>, ool_output_size=<unavailable>) + 499 at IOUserClient.cpp:3489
    frame #5: 0xffffff8020dea517 kernel`_Xio_connect_method(InHeadP=0xffffff802c24b600, OutHeadP=0xffffff80348915d0) + 391 at device_server.c:8249
    frame #6: 0xffffff8020d3e91c kernel`ipc_kobject_server(request=0xffffff802f2d7000) + 252 at ipc_kobject.c:338
    frame #7: 0xffffff8020d235a3 kernel`ipc_kmsg_send(kmsg=<unavailable>, option=<unavailable>, send_timeout=0) + 291 at ipc_kmsg.c:1430
    frame #8: 0xffffff8020d33e8d kernel`mach_msg_overwrite_trap(args=<unavailable>) + 205 at mach_msg.c:487
    frame #9: 0xffffff8020e0a142 kernel`mach_call_munger64(state=0xffffff80317ff560) + 386 at bsd_i386.c:542

```

Below is an example of a call stack when data is received from network

```
    frame #0: 0xffffff7fa355ee80 NetworkKernelExtension`NkeSocketFilter::FltDataIn(cookie=0xffffff8034cc4200, so=0xffffff8035429a70, from=0x0000000000000000, data=0xffffff80b9ebb6c8, control=0x0000000000000000, flags=0) + 32 at NkeSocketFilter.cpp:835
    frame #1: 0xffffff802123d754 kernel`sflt_data_in(so=0xffffff8035429a70, from=0x0000000000000000, data=0xffffff80b9ebb6c8, control=0x0000000000000000, flags=0) + 276 at kpi_socketfilter.c:1191
    frame #2: 0xffffff80212215ce kernel`sbappendstream(sb=0xffffff8035429ae0, m=0xffffff80b5663100) + 174 at uipc_socket2.c:800
    frame #3: 0xffffff802107ff6b kernel`tcp_input(m=<unavailable>, off0=<unavailable>) + 16283 at tcp_input.c:2666
    frame #4: 0xffffff8021071209 kernel`ip_proto_dispatch_in(m=0xffffff80b5663100, hlen=<unavailable>, proto=<unavailable>, inject_ipfref=<unavailable>) + 361 at ip_input.c:635
    frame #5: 0xffffff8021071579 kernel`ip_input(m=0xffffff80b5663100) + 761 at ip_input.c:1293
    frame #6: 0xffffff802105d1c1 kernel`ip_proto_input(protocol=<unavailable>, packet_list=0x0000000000000000) + 33 at in_proto.c:336
    frame #7: 0xffffff8020fd68a2 kernel`proto_input(protocol=<unavailable>, packet_list=0xffffff80b5663100) + 162 at kpi_protocol.c:276
    frame #8: 0xffffff8020fb1ec4 kernel`ether_inet_input(ifp=<unavailable>, protocol_family=<unavailable>, m_list=0xffffff80b5663100) + 756 at ether_inet_pr_module.c:219
    frame #9: 0xffffff8020fad87e kernel`dlil_input_packet_list_common [inlined] dlil_ifproto_input(m=0xffffff80b5663100) + 27 at dlil.c:3160
    frame #10: 0xffffff8020fad863 kernel`dlil_input_packet_list_common(ifp_param=0x0000000000000000, m=0xffffff80b5663100, cnt=<unavailable>, mode=<unavailable>, ext=<unavailable>) + 3507 at dlil.c:3450
    frame #11: 0xffffff8020fafec9 kernel`dlil_input_thread_func [inlined] dlil_input_packet_list_extended(ifp=<unavailable>, m=<unavailable>, cnt=<unavailable>, mode=<unavailable>) + 265 at dlil.c:3274
    frame #12: 0xffffff8020fafebe kernel`dlil_input_thread_func(v=0xffffff802d931328, w=<unavailable>) + 254 at dlil.c:1873
```

Then the filter notifies a usermode client and waits in `NkeSocketObject::FltData` for a response from the client

```
errno_t	
NkeSocketObject::FltData(
    __in socket_t so,
    __in const struct sockaddr *addr, // a peer's address
    __inout mbuf_t *data,
    __inout mbuf_t *control,
    __in sflt_data_flag_t flags,
    __in NkeSocketDataDirection  direction
    )
{
....

                    error = userClient->socketFilterNotification( &notification );
                    
....

            while( wait && (! waitEntry.waitSatisfied ) ){
                
                assert( ! sendNotification );
                
                struct timespec ts = { 1, 0 };       // one second
                
                (void)msleep( &waitEntry,                   // wait channel
                              NULL,                         // mutex
                              PUSER,                        // priority
                              "NkeSocketObject::FltData()", // wait message
                              &ts );                        // sleep interval
            } // while( wait && (! waitEntry.waitSatisfied ) )
....
}
```

Similarly a synchronous processing can be implemented for other callbacks.


##Filter loading

The filter module is loaded by kextload command. The user client connects to the filter IOKit object to receive events and process data.
The filter blocks connections until a client is connected.

```
mac$ sudo kextload ./NetworkKernelExtension.kext
mac$ ./NkeClient
```

