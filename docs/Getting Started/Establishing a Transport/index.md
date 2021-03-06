# Establishing a Transport

SDL is theoretically transport agnostic, meaning it can work over any potential transport. However, due to limitations of app platforms, only certain transports are available for connecting to given app platforms.

The first step in using SDL is to establish a connection over a primary transport.

![Step 1: Establish a transport connection](assets/overall_1.png) 



## Primary and Secondary Transports

SDL has the ability to register a session across multiple transports. This is useful for certain scenarios where a higher bandwidth transport is needed to perform more data intensive operations of SDL (such as video/audio streaming), but that transport isn't good for establishing or maintaining the connection.

### Android

#### Bluetooth

Primary or secondary transport

Bluetooth connections established to an Android device require a few prerequisites and processes. First, the bluetooth's Advanced Audio Distribution Profile (A2DP) and Hands Free Profile (HFP) must be connected for the SDL Android router service to start listening on an RFCOMM channel using the Serial Port Profile (SPP). The Android device will use a router service that allows multiple apps to bind to it and utilize a single transport connection. This allows for a greater chance that the SDL library can obtain one of the RFCOMM channels available on the device, as most devices have rather low limits. In some cases, apps might not bind to the router service and listen on their own bluetooth transport. These apps must also be connected.

![Android bluetooth process](assets/android_bluetooth.png) 

##### Connection Timing

- After the initial connection of A2DP and HFP, the head unit will only have a limited time to connect to the SDL RFCOMM channel. This is due to several Android operating system restrictions and requirements.
- The router service must be started as a foreground service which causes a notification to appear on the users phone. The library makes adjustments to the time the router service spends in the foreground based on three scenarios: 
    1. **The router service has never seen this hardware before:** In this case the router service will start in the foreground and remain there for around 10 seconds. If the RFCOMM channel _does_ establish a connection during this time, the router service will remain in the foreground. However, if the RFCOMM channel _does not_ establish a connection, the service will exit from the foreground after the timeout has been reached. 
    2.  **The router service has seen this hardware before, but has never connected:** In this case router service will immediately exit from the foreground after being started.
    3. **The router service has successfully connected to this hardware before:** The router service will remain in foreground for around 20 seconds. If the RFCOMM channel _does_ establish a connection during this time, the router service will remain in the foreground. However, if the RFCOMM channel _does not_ establish a connection, the service will exit from the foreground after the timeout has been reached. 
- If a connection is not established during any of the timeout periods and the service exits the foreground, the service does not typically close immediately. It stays in the background until the Android operating system closes it. This timing is not guaranteed or consistent between devices, and therefore the service should be considered "closed" after this timeout occurs.
- It is possible for the router service to be started after the timeout and close from initial bluetooth connection, if an app is launched with the correct integration it should call a method that potentially restarts the router service (similar to `queryForConnectedService(context)`). However, the hardware will have to perform an SDP query to know that this has occurred. 

##### Number of RFCOMM Channels
- In most cases, the first SDP query will only return a single RFCOMM channel for the SDL UUID; this is the router service's RFCOMM channel. After connecting to that RFCOMM channel it is recommended to do another SDP query. This is due to some apps not trusting the previously connected router service, and they will start their own Bluetooth transport with a different RFCOMM channel but still use the same SDL UUID. The head unit should connect to each additional RFCOMM channel with the SDL UUID.
- Due to the previous bullet or other situations (such as if an app crashes), it is also recommend an SDP query be triggered at some other point. Common triggers are user requested button presses, transport disconnection, or app unregistration.
- There is not guarantee on which RFCOMM channel with the SDL UUID is the router service. If during the first SDP query multiple RFCOMM channels are found with the SDL UUID, each one of them will need to be connected in order to ensure that the router service is connected and all other apps connect as well.

#### USB

Primary or secondary transport

Android has a specification called AOA that allows apps to communicate over a USB connection. The default implementation of AOA on the Android device is very limited. Users must select an app to receive the "USB device" upon connection on the device. However, the SDL Android library leverages its already existing functionality of multiplexing a single transport connection through a router service to allow multiple apps to use the AOA connection. 

![Android AOA process](assets/android_aoa.png) 

##### Limitations
- AOA has been part of the SDL Android library for a while, however, the ability to multiplex the single AOA connection has only been part of the library since SDL Android Library version 4.7.0 and newer. This means that some apps might still have the older integration that does not support AOA through the router service.
- The library is designed to handle this case as long as the older AOA app updates their library and either updates its integration to support the router service or another app has the newer library that supports the router service.
- The AOA connection is only given to an app that the user selects on the device at the time of connecting the USB cable. This means that the user has to select one of the apps that can support multiplexing the AOA connection for other apps to connect. If a user selects an app that _doesn't_ support multiplexing over AOA, only that app will register. 

#### TCP
Secondary transport only

Due to limitations for the iOS platform (discussed in the next section), Android can only use the TCP transport as a secondary transport in production. The hardware will send the connection information after a primary transport is connected through an SDL packet.

### iOS

#### Bluetooth and USB
Primary transport only

Apple has a proprietary technology called iAP that apps connect through to communicate data over Bluetooth or USB. Unfortunately, this means that none of that information can be shared here. You will have to build and certify your hardware as part of [Apple's MFi program](https://developer.apple.com/programs/mfi/) in order to enable the Bluetooth and USB transports, which are also the only primary transport available. Therefore, without MFi certification and building support for the iAP transport, your hardware cannot support production iOS devices and apps.

##### Single-Session vs. Multi-Session Connection Schemes

The way IAP works is that you have a protocol string that a hardware device and an app declare that they can use. When iOS sees a hardware device with a known protocol string, it notifies all apps that declare that protocol string. The app can then open a 1-1 connection with the hardware using that protocol string as an identifier for the connection. 

###### Single-Session Connections (Legacy)


There are two different connection strategies that the iOS library uses. The first is the "hub" strategy for connecting to a single-session iAP protocol. This method is only included in the library as legacy support and should not be implemented in new modules. The hub strategy was developed to allow multiple apps to connect to the hardware before multiple apps could use the same protocol string.

The hardware has to make 30 protocol strings available. Each app will attempt to connect to one protocol (`prot0`, called the control protocol) in order to receive one byte of data. That data is a number 1-29. The app will then disconnect from the control protocol and connect to a data protocol corresponding to the number they received (prot1-prot29).

Because many apps are attempting to connect to the control protocol in order to receive their protocol string number, the app is given a timeout based on a hash of the app’s name. The apps will try to connect to the control string after that timeout. If they fail, they’ll wait the timeout 3 more times and attempt to connect to get their data protocol number.

![Single app with hub strategy](assets/ios_hub_single_app.png) 



This strategy works, but has some large downsides. The more apps that try to connect to the control protocol, the more collisions will result, and the less reliable the connection becomes. Once more than four or five apps connect, the connection will degrade quickly and some apps may connect sometimes and not other times.


![Multi app with hub strategy](assets/ios_hub_multi_app.png) 


##### Multi-Session Connections (Modern)


Multi-Session is a newer iAP feature that allows multiple connections to a single protocol string. Every app can connect to the multi-session protocol string without needing to wait or needing to manage connection timing. This is the preferred method to connect iOS SDL apps because it eliminates a lot of the stability issues that have been observed with the single-session method.  See [this evolution proposal](https://github.com/smartdevicelink/sdl_evolution/blob/master/proposals/0080-Support-for-MultiSession-protocol-string.md) for its acceptance into the project.


![Multisession approach](assets/ios_multisession.png) 


#### TCP
Secondary transport only

iOS has a very strict limitation that no TCP connection can be maintained while an app is in the background. For this reason TCP can only be a secondary transport and should only be used for video streaming applications, which already have this limitation. The hardware will send the connection information to the library after a primary transport is connected through an SDL packet, and the library will then take care of attempting to connect to the IP address and port.


### Java SE

#### Websocket Server
Primary transport only

The only transport currently available by default in the SDL Java SE library is a web socket server connection. The Java SE app should usually run directly on the same device as the SDL Core and HMI (i.e. it should be an embedded application).

Determining when the Java SE app should start listening to that web socket port will need to be defined by the SDL Core integrator. This will likely be during start up of the device. The Java SE app can simply be started and wait for the user to select the application from the application list. See the [0158 - Cloud App Transport Adapter](https://github.com/smartdevicelink/sdl_evolution/blob/master/proposals/0158-cloud-app-transport-adapter.md) proposal for more information.

![Socket connection](assets/socket.png) 

#### Secondary Transport

Currently the Java SE client library does not support a secondary transport because the WebSocket transport is already high bandwidth.

### Java EE

#### Java Beans (Custom Transport)
Primary Transport Only

Java EE is usually built around Java Beans in order to easily handle many simultaneous connections that each have their own, independent state. This is another example of a proprietary or separately licensed transport. Because of this, the Java EE library has what's called a Custom Transport. This also developers to hook into the Java EE Bean framework without the SDL library having to include that code into the library itself.
Java EE apps are designed to be long running applications that live on an off-board server. The SDL Core implementation will connect out to these applications once they are selected from the app list on the HMI. See the [0158 - Cloud App Transport Adapter](https://github.com/smartdevicelink/sdl_evolution/blob/master/proposals/0158-cloud-app-transport-adapter.md) proposal for more information.

The previous flow for JavaSE can be followed as the difference will depend on the type of custom transport used. For most cases, using Java Beans is the best flow and will only differ in adding multiple connections to the app via independent instances created by a type of load balancer. 

#### Secondary Transport

Currently the Java EE client library does not support a secondary transport as the websocket transport is already high bandwidth.
