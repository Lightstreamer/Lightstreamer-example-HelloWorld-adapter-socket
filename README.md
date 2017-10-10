# Lightstreamer - "Hello World" Tutorial - TCP Sockets Adapter #
<!-- START DESCRIPTION lightstreamer-example-helloworld-adapter-socket -->

This is the third project in a series aimed at illustrating how to develop Lightstreamer Adapters based on various technologies:

- [Lightstreamer - "Hello World" Tutorial - Java Adapter](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-java): An introduction to Lightstreamer's data model, Java Data Adapters, and Server deployment, through the development of a very basic "Hello World" application.
- [Lightstreamer - "Hello World" Tutorial - .NET Adapter](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-dotnet): The .NET version of the Data Adapter used in the "Hello World" application, showing both a C# and a Visual Basic port.

In this third installment, we'll show a *Data Adapter* that communicates with the Lightstreamer Server through plain <b>TCP sockets</b>, instead of leveraging higher level abstractions as did with the Java API and the .NET API.

The rationale for this is to enable the development of Data Adapters based on technologies other than Java and .NET. This way, it is possible to inject real-time updates into Lightstreamer Server from programs written in <b>C, PHP, Ruby,</b> or any other language that allows client <b>TCP socket programming</b>.

As an example of [Clients Using This Adapter](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket#clients-using-this-adapter), you may refer to the [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) and view the corresponding [Live Demo](http://demos.lightstreamer.com/HelloWorld/).

## Details

### The Architecture

On the client side, we will keep the same exact HTML front-end used in the two previous installments. For an explanation of the HTML/JavaScript code, please see the [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) project.

On the *server side*, we will leverage the [Lightstreamer Adapter Remoting Infrastructure (ARI)](http://www.lightstreamer.com/docs/adapter_generic_base/ARI%20Protocol.pdf):

![General architecture](general_architecture.png)

This is the same architecture used in [Lightstreamer - "Hello World" Tutorial - .NET Adapter](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-dotnet), but in this case, the Remote Data Adapter is not a .NET application, but any process that opens two sockets with the Proxy Data Adapter and implements the ARI Protocol over TCP. As in the previous examples, we will not code a custom Metadata Adapter, but will use a default one. So the Remote Metadata Adapter will not be present.

Let's recap. On the server side, this new example will be comprised of:

- **Proxy Data Adapter**: A ready-made Data Adapter, based on Java, which is provided as part of the Adapter Remoting Infrastructure (available in the free and commercial distributions of Lightstreamer Server).
- **Remote Data Adapter**: The subject of this article...
- **LiteralBasedProvider**: A ready-made Metadata Adapter, based on Java, which is provided as part of all the free and commercial distributions of Lightstreamer Server. The LiteralBasedProvider replaces the Proxy Metadata Adapter in the diagram above.

So, what about the *Remote Data Adapter*? We could implement it in any language that supports socket programming, but in this article, we will do something more interactive, which does not require any programming. We will use a very normal telnet client to connect to the Proxy Data Adapter and will manually play the ARI Network Protocol.
Should be fun...

The Proxy Data Adapter listens on two TCP ports and the Remote Data Adapter has to create two sockets. One socket is used for interactions based on a **request/response** paradigm (this is a *synchronous channel*). The other socket is used to deliver **asynchronous events** from the Remote Adapter to the Proxy Adapter (this is an *asynchronous channel*). Therefore, our Remote Data Adapter will be comprised of two telnet windows.
<!-- END DESCRIPTION lightstreamer-example-helloworld-adapter-socket -->

### The Network Protocol

In the examples, we scratch the surface of the ARI Network Protocol. By delving into deeper details, you will see that it is quite straightforward. The full specification is available in the [ARI Protocol.pdf document](http://www.lightstreamer.com/docs/adapter_generic_base/ARI%20Protocol.pdf).

The Remote Data Adapter can only receive two synchronous requests: `subscribe` and `unsubscribe`. It can send three asynchronous events: `update`, `end of snapshot`, and `failure`.
The Remote Metadata Adapter (which is not covered in this article) can receive more synchronous requests, as its interface is a bit more complex than the Data Adapter, but it does not send any asynchronous events at all (in fact, it uses one TCP socket only).

In reality, of course, you will never implement a "human-driven" Adapter as we did in this article, but this approach is useful, from a didactive perspective, to illustrate the basic principles of the Lightstreamer Adapter Remoting Infrastructure (ARI). 
If you need to develop an Adapter based on technologies other than Java, .NET, and Node.js [for which a higher level interface is available](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket#related-projects) the ARI makes the trick.

Should you develop any Adapter in <b>PHP, Ruby, Python, Perl</b>, or any other language, feel free to let us know, by posting a comment here on the [Lightstreamer Forums](http://forums.lightstreamer.com/).

### Dig the Code

#### The Adapter Set Configuration

This Adapter Set Name is configured and will be referenced by the clients as `PROXY_HELLOWORLD_SOCKETS`.
For this demo, we configure just the Data Adapter as a *Proxy Data Adapter*, while instead, as Metadata Adapter, we use the [LiteralBasedProvider](https://github.com/Lightstreamer/Lightstreamer-example-ReusableMetadata-adapter-java), a simple full implementation of a Metadata Adapter, already provided by Lightstreamer server.
As *Proxy Data Adapter*, you may configure also the robust versions. The *Robust Proxy Data Adapter* has some recovery capabilities and avoid to terminate the Lightstreamer Server process, so it can handle the case in which a Remote Data Adapter is missing or fails, by suspending the data flow and trying to connect to a new Remote Data Adapter instance. Full details on the recovery behavior of the Robust Data Adapter are available as inline comments within the `DOCS-SDKs/adapter_remoting_infrastructure/doc/adapter_robust_conf_template/adapters.xml` file in your Lightstreamer Server installation.

The `adapters.xml` file for this demo should look like:
```xml
<?xml version="1.0"?>
<adapters_conf id="PROXY_HELLOWORLD_SOCKETS">
 
  <metadata_provider>
    <adapter_class>com.lightstreamer.adapters.metadata.LiteralBasedProvider</adapter_class>
  </metadata_provider>
 
  <data_provider>
    <adapter_class>PROXY_FOR_REMOTE_ADAPTER</adapter_class>
    <classloader>log-enabled</classloader>
      <param name="request_reply_port">7001</param>
      <param name="notify_port">7002</param>
      <param name="timeout">36000000</param>
  </data_provider>
 
</adapters_conf>
```
We have chosen TCP port <b>7001</b> for the request/response channel and TCP port <b>7002</b> for the asynchronous channel. Feel free to use different ports if such numbers are already used on your system.

Notice the `timeout` parameter. It sets the maximum time the Proxy Adapter will wait for a response from the Remote Adapter after issuing a request. The default value is 10 seconds, but in this case, the Remote Adapter is played by humans, so we have configured a very high value (10 hours), to do relaxed experiments without the pressure of any timeouts.

<i>NOTE: not all configuration options of a Proxy Adapter are exposed by the file suggested above.<br>
You can easily expand your configurations using the generic template, `DOCS-SDKs/adapter_remoting_infrastructure/doc/adapter_conf_template/adapters.xml` or `DOCS-SDKs/adapter_remoting_infrastructure/doc/adapter_robust_conf_template/adapters.xml`, as a reference.</i>

## Install
If you want to install a version of this demo in your local Lightstreamer Server, follow these steps:
* Download *Lightstreamer Server* (Lightstreamer Server comes with a free non-expiring demo license for 20 connected users) from [Lightstreamer Download page](http://www.lightstreamer.com/download.htm), and install it, as explained in the `GETTING_STARTED.TXT` file in the installation home directory.
* Get the `deploy.zip` file for the Lightstreamer version you have installed from [releases](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket/releases) and unzip it, obtaining the `deployment` folder.
* Plug the Proxy Data Adapter into the Server: go to the `deployment/Deployment_LS` folder and copy the `SocketHelloWorld` directory and all of its files to the `adapters` folder of your Lightstreamer Server installation.
* Alternatively, you may plug the *robust* versions of the Proxy Data Adapter: go to the `deployment/Deployment_LS(robust)` folder and copy the `SocketHelloWorld` directory and all of its files into `adapters` folder of your Lightstreamer Server installation. 
* Install the [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) listed in [Clients Using This Adapter](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-node#clients-using-this-adapter).
    * To make the ["Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) front-end pages get data from the newly installed Adapter Set, you need to modify the front-end pages and set the required Adapter Set name to PROXY_HELLOWORLD_SOCKETS, when creating the LightstreamerClient instance. So edit the `index.htm` page of the Hello World front-end, deployed under `Lightstreamer/pages/HelloWorld`, and replace:<BR/>
`var client = new LightstreamerClient(null," HELLOWORLD");`<BR/>
with:<BR/>
`var client = new LightstreamerClient(null,"PROXY_HELLOWORLD_SOCKETS");;`<BR/>
Add the .setRequestedSnapshot("yes") line to always get the current state of the fields

### Run the Demo

* Launch Lightstreamer Server from a command or shell window (which we will call the *"log window"*). The Server will wait for the "PROXY_HELLOWORLD_SOCKETS" Remote Data Adapter to connect to the two TCP ports we specified above (7001 and 7002), and the Server startup will complete only after a successful connection between the Proxy Data Adapter and the Remote Data Adapter. You should see something like this:
```cmd
[...]
30.ott.13 17:07:22,227 < INFO> Lightstreamer Server 6.0 a1 build 1650
30.ott.13 17:07:22,286 < INFO> Lightstreamer Server starting in Moderato edition.
30.ott.13 17:07:22,346 < WARN> Only minimal JMX management support is available with the current license.
30.ott.13 17:07:22,444 < INFO> Started RMI server for JMX on port 8888.
30.ott.13 17:07:22,525 < INFO> Bound RMI Connector for JMX on port 8888 (communication on port 8888).
30.ott.13 17:07:22,568 < INFO> Bound RMI Connector for Platform mbeans on port 8888 (communication on port 8888).
30.ott.13 17:07:22,574 < INFO> SERVER pool size set by default at 10.
30.ott.13 17:07:22,696 < INFO> data_provider element without name attribute; using DEFAULT as the default name.
30.ott.13 17:07:22,699 < INFO> Loading Data Adapter PROXY_HELLOWORLD_SOCKETS.DEFAULT
30.ott.13 17:07:22,699 < INFO> Loading Metadata Adapter PROXY_HELLOWORLD_SOCKETS
30.ott.13 17:07:22,706 < INFO> Finished loading Metadata Adapter PROXY_HELLOWORLD_SOCKETS
30.ott.13 17:07:22,736 < INFO> Connecting...
30.ott.13 17:07:22,740 < INFO> Waiting for a connection on port 7002...
30.ott.13 17:07:22,741 < INFO> Waiting for a connection on port 7001...
.
```
* Open a command or a shell windows (which I will call the *"request/response window"*) and type :
```cmd
  telnet localhost 7001
```
* Open a command or a shell windows (which I will call the *"async window"*) and type:
```cmd
  telnet localhost 7002
```
Promptly, the Server will try to initialize our Remote Data Adapter; this will cause a request to be issued in the request/response window, similar to the following:
```cmd
  10000014209a460b9|DPI|S|data_provider.name|S|DEFAULT|S|adapters_conf.id|S|PROXY_HELLOWORLD_SOCKETS
```
*Note: The first string "10000014209a460b9" is the unique ID of that request and will actually change every time. We will use the actual ID we received for the reply message.*
* Let's respond saying that we accept such subscription. We can do this by typing the following string in the request/response window and hitting Enter:
```cmd
  10000014209a460b9|DPI|V
```
*Note: Replace "10000014209a460b9" with the actual ID you received, otherwise the initialization will not succeed and you will see a warning in the log window.*

The Server initialization will complete and in the log window, you should see something like this:
```cmd
[...]
30.ott.13 17:07:22,227 < INFO> Lightstreamer Server 6.0 a1 build 1650
30.ott.13 17:07:22,286 < INFO> Lightstreamer Server starting in Moderato edition.
30.ott.13 17:07:22,346 < WARN> Only minimal JMX management support is available with the current license.
30.ott.13 17:07:22,444 < INFO> Started RMI server for JMX on port 8888.
30.ott.13 17:07:22,525 < INFO> Bound RMI Connector for JMX on port 8888 (communication on port 8888).
30.ott.13 17:07:22,568 < INFO> Bound RMI Connector for Platform mbeans on port 8888 (communication on port 8888).
30.ott.13 17:07:22,574 < INFO> SERVER pool size set by default at 10.
30.ott.13 17:07:22,696 < INFO> data_provider element without name attribute; using DEFAULT as the default name.
30.ott.13 17:07:22,699 < INFO> Loading Data Adapter PROXY_HELLOWORLD_SOCKETS.DEFAULT
30.ott.13 17:07:22,699 < INFO> Loading Metadata Adapter PROXY_HELLOWORLD_SOCKETS
30.ott.13 17:07:22,706 < INFO> Finished loading Metadata Adapter PROXY_HELLOWORLD_SOCKETS
30.ott.13 17:07:22,736 < INFO> Connecting...
30.ott.13 17:07:22,740 < INFO> Waiting for a connection on port 7002...
30.ott.13 17:07:22,741 < INFO> Waiting for a connection on port 7001...
30.ott.13 17:07:47,368 < INFO> Connected on port 7001
30.ott.13 17:08:02,448 < INFO> Connected on port 7002
30.ott.13 17:08:02,450 < INFO> Connected
30.ott.13 17:08:02,451 < INFO> Finished loading Data Adapter PROXY_HELLOWORLD_SOCKETS.DEFAULT
30.ott.13 17:08:02,465 < INFO> Selector pool size set by default at 8.
30.ott.13 17:08:02,466 < INFO> Selector maximum load set by default at 1000.
30.ott.13 17:08:02,758 < INFO> Notify receiver '#1' starting...
30.ott.13 17:08:02,763 < INFO> Request sender '#1' starting...
30.ott.13 17:08:02,764 < INFO> Events pool size set by default at 8.
30.ott.13 17:08:02,765 < INFO> Reply receiver '#1' starting...
30.ott.13 17:08:02,779 < INFO> Pump pool size set by default at 8.
30.ott.13 17:08:02,795 < INFO> Lightstreamer on Java Virtual Machine: Sun Microsystems Inc., Java HotSpot(TM) 64-Bit Server VM, 20.5-b03, 1.6.0_30-b12 on Windows 7
30.ott.13 17:08:02,796 < INFO> Lightstreamer Server 6.0 a1 build 1650 starting...
30.ott.13 17:08:02,811 < INFO> Server "Lightstreamer HTTP Server" listening to *:8080 ....
```
* Open a browser window and go to: [http://localhost:8080/HelloWorld/]()
* In the *log window*, you will see some information regarding the HTTP interaction between the browser and the Lightstreamer Server.
* In the browser window, you will see:
```cmd
    loading...
    loading...
```
* The `greetings` item has been subscribed too by the Client, with a schema comprised of the `message` and `timestamp` fields. The Server has then subscribed to the same item through our Remote Adapter (because of the fact that Lightstreamer Server is based on a "Publish On-Demand" paradigm). This subscription will manifest itself as a request in the *request/response window*, similar to the following:
```cmd
  20000014209a460b9|SUB|S|greetings
```
*Note: The first string "20000014209a460b9" is the unique ID of that request and will actually change every time. We will use the actual ID we received for the reply message and the messages to be published.*
* Let's respond saying that we accept such subscription. We can do this by typing the following string in the *request/response window* and hitting Enter:
```cmd
  20000014209a460b9|SUB|V
```
*Note: Replace "20000014209a460b9" with the actual ID you received.*
* Our Remote Data Adapter has now accepted to serve events on the `greetings` item. It's time to inject some events by hand, through the *async window*. With most telnet applications, you will not see anything when typing in the *async window*, so it is better to use copy and paste. Paste the following string, then hit Enter:
```cmd
  0|UD3|S|greetings|S|20000014209a460b9|B|0|S|timestamp|S|Now is the time|S|message|S|Hello socket world!
```
*Note: Make sure to paste everything on a single line, and replace "20000014209a460b9" with the actual ID you received.*
* Now look at the browser window and enjoy the results of this effort:
```cmd
  Hello socket world!
  Now is the time
```
* We can push more events on the `greetings` item, leveraging the same two fields `message` and `timestamp`, ans sending arbitrary data. For example, paste this in the *async window* (always on a single line and replacing the ID):
```cmd
  0|UD3|S|greetings|S|20000014209a460b9|B|0|S|message|S|What do you call a fish with no eyes?|S|timestamp|S|A fsh
```

## See Also

### Clients Using This Adapter

<!-- START RELATED_ENTRIES -->

* [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript)

<!-- END RELATED_ENTRIES -->

### Related Projects

* [Complete list of "Hello World" Adapter implementations with other technologies](https://github.com/Lightstreamer?utf8=%E2%9C%93&q=Lightstreamer-example-HelloWorld-adapter&type=&language=)
* [Lightstreamer - Reusable Metadata Adapters - Java Adapter](https://github.com/Lightstreamer/Lightstreamer-example-ReusableMetadata-adapter-java)

## Lightstreamer Compatibility Notes

- Compatible with Lightstreamer SDK for Generic Adapters version 1.7 or newer.
- Compatible with Lightstreamer JavaScript Client Library version 6.0 or newer.
- For a version of this example compatible with SDK for Generic Adapters 1.4.3, please refer to [this tag](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket/tree/for_Lightstreamer_5.1).

## Final Notes
For more information, please [visit our website](http://www.lightstreamer.com/) and [post to our support forums](http://forums.lightstreamer.com) any feedback or questions you might have. Thanks!
