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

On the *server side*, we will leverage the [Lightstreamer Adapter Remoting Infrastructure (ARI)](https://lightstreamer.com/api/ls-generic-adapter/latest/ARI%20Protocol.pdf):

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

In the examples, we scratch the surface of the ARI Network Protocol. By delving into deeper details, you will see that it is quite straightforward. The full specification is available in the [ARI Protocol.pdf document](https://lightstreamer.com/api/ls-generic-adapter/latest/ARI%20Protocol.pdf).

The Remote Data Adapter can only receive two synchronous requests: `subscribe` and `unsubscribe`. It can send three asynchronous events: `update`, `end of snapshot`, and `failure`.
The Remote Metadata Adapter (which is not covered in this article) can receive more synchronous requests, as its interface is a bit more complex than the Data Adapter, but it does not send any asynchronous events at all (in fact, it uses one TCP socket only).

In reality, of course, you will never implement a "human-driven" Adapter as we did in this article, but this approach is useful, from a didactive perspective, to illustrate the basic principles of the Lightstreamer Adapter Remoting Infrastructure (ARI). 
If you need to develop an Adapter based on technologies other than Java, .NET, and Node.js [for which a higher level interface is available](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket#related-projects) the ARI makes the trick.

Should you develop any Adapter in <b>PHP, Ruby, Python, Perl</b>, or any other language, feel free to let us know, by posting a comment here on the [Lightstreamer Forums](http://forums.lightstreamer.com/).

### Dig the Code

#### The Adapter Set Configuration

This Adapter Set Name is configured and will be referenced by the clients as `PROXY_HELLOWORLD_SOCKETS`.
For this demo, we configure just the Data Adapter as a *Proxy Data Adapter*, while instead, as Metadata Adapter, we use the [LiteralBasedProvider](https://github.com/Lightstreamer/Lightstreamer-example-ReusableMetadata-adapter-java), a simple full implementation of a Metadata Adapter, already provided by Lightstreamer server.
As *Proxy Data Adapter*, you may configure also the robust versions. The *Robust Proxy Data Adapter* has some recovery capabilities and avoid to terminate the Lightstreamer Server process, so it can handle the case in which a Remote Data Adapter is missing or fails, by suspending the data flow and trying to connect to a new Remote Data Adapter instance. Full details on the recovery behavior of the Robust Data Adapter are available as inline comments within the [provided template](https://lightstreamer.com/docs/ls-ARI/latest/adapter_robust_conf_template/adapters.xml).

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
You can easily expand your configurations using the generic template
for [basic](https://lightstreamer.com/docs/ls-ARI/latest/adapter_conf_template/adapters.xml) and [robust](https://lightstreamer.com/docs/ls-ARI/latest/adapter_robust_conf_template/adapters.xml) Proxy Adapters as a reference.</i>

## Install
If you want to install a version of this demo in your local Lightstreamer Server, follow these steps:
* Download *Lightstreamer Server* (Lightstreamer Server comes with a free non-expiring demo license for 20 connected users) from [Lightstreamer Download page](http://www.lightstreamer.com/download.htm), and install it, as explained in the `GETTING_STARTED.TXT` file in the installation home directory.
* Get the `deploy.zip` file of the [latest release](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket/releases) and unzip it, obtaining the `deployment` folder.
* Plug the Proxy Data Adapter into the Server: go to the `deployment/Deployment_LS` folder and copy the `SocketHelloWorld` directory and all of its files to the `adapters` folder of your Lightstreamer Server installation.
* Alternatively, you may plug the *robust* versions of the Proxy Data Adapter: go to the `deployment/Deployment_LS(robust)` folder and copy the `SocketHelloWorld` directory and all of its files into `adapters` folder of your Lightstreamer Server installation. 
* Install the [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) listed in [Clients Using This Adapter](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket#clients-using-this-adapter).
    * To make the ["Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) front-end pages get data from the newly installed Adapter Set, you need to modify the front-end pages and set the required Adapter Set name to PROXY_HELLOWORLD_SOCKETS, when creating the LightstreamerClient instance. So edit the `index.htm` page of the Hello World front-end, deployed under `Lightstreamer/pages/HelloWorld`, and replace:<BR/>
`var client = new LightstreamerClient(null," HELLOWORLD");`<BR/>
with:<BR/>
`var client = new LightstreamerClient(null,"PROXY_HELLOWORLD_SOCKETS");;`<BR/>
Also add the .setRequestedSnapshot("yes") line to always get the current state of the fields. The result is shown in the sample file for Web Client Library 8.0.0.


### Run the Demo

* Launch Lightstreamer Server from a command or shell window (which we will call the *"log window"*). The Server will wait for the "PROXY_HELLOWORLD_SOCKETS" Remote Data Adapter to connect to the two TCP ports we specified above (7001 and 7002), and the Server startup will complete only after a successful connection between the Proxy Data Adapter and the Remote Data Adapter. You should see something like this:
```cmd
[...]
23-gen-19 15:52:17,241 < INFO> Lightstreamer Server 7.1.0 b4 build 1913
23-gen-19 15:52:17,248 < INFO> Server launched on Java Virtual Machine: Oracle Corporation, Java HotSpot(TM) 64-Bit Server VM, 25.66-b18, 1.8.0_66-b18 on Windows 10
23-gen-19 15:52:18,196 < WARN> Lightstreamer Server is running with a Demo license, which has a limit of 20 concurrent users and can be used for evaluation, development, and testing, but not for production. If you need to evaluate Lightstreamer Server without this user limit, or need any information on the other license types, please contact info@lightstreamer.com
23-gen-19 15:52:20,700 < INFO> Number of detected cores: 4
23-gen-19 15:52:20,701 < INFO> Lightstreamer Server starting in ENTERPRISE edition.
23-gen-19 15:52:34,054 < INFO> Pump pool size set by default at 4.
23-gen-19 15:52:35,004 < INFO> Events pool size set by default at 4.
23-gen-19 15:52:35,076 < INFO> Snapshot pool size set by default at 10.
23-gen-19 15:52:37,702 < INFO> Loading Data Adapter PROXY_HELLOWORLD_SOCKETS.DEFAULT
23-gen-19 15:52:37,703 < INFO> Loading Metadata Adapter for Adapter Set PROXY_HELLOWORLD_SOCKETS
23-gen-19 15:52:38,763 < INFO> Finished loading Metadata Adapter PROXY_HELLOWORLD_SOCKETS
23-gen-19 15:52:39,704 < INFO> Connecting...
23-gen-19 15:52:39,817 < INFO> Waiting for a connection on port 7002...
23-gen-19 15:52:39,819 < INFO> Waiting for a connection on port 7001...
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
  10000014209a460b9|DPI|S|data_provider.name|S|DEFAULT|S|adapters_conf.id|S|PROXY_HELLOWORLD_SOCKETS|S|ARI.version|S|1.8.2
```
*Note: The first string "10000014209a460b9" is the unique ID of that request and will actually change every time. We will use the actual ID we received for the reply message.*
* Let's respond saying that we accept such initialization parameters. We can do this by typing the following string in the request/response window and hitting Enter:
```cmd
  10000014209a460b9|DPI|S|ARI.version|S|1.8.2
```
where we have also agreed on version 1.8.2 of the protocol.
*Note: Replace "10000014209a460b9" with the actual ID you received, otherwise the initialization will not succeed and you will see a warning in the log window.*

The Server initialization will complete and in the log window, you should see something like this:
```cmd
[...]
23-gen-19 15:52:17,241 < INFO> Lightstreamer Server 7.1.0 b4 build 1913
23-gen-19 15:52:17,248 < INFO> Server launched on Java Virtual Machine: Oracle Corporation, Java HotSpot(TM) 64-Bit Server VM, 25.66-b18, 1.8.0_66-b18 on Windows 10
23-gen-19 15:52:18,196 < WARN> Lightstreamer Server is running with a Demo license, which has a limit of 20 concurrent users and can be used for evaluation, development, and testing, but not for production. If you need to evaluate Lightstreamer Server without this user limit, or need any information on the other license types, please contact info@lightstreamer.com
23-gen-19 15:52:20,700 < INFO> Number of detected cores: 4
23-gen-19 15:52:20,701 < INFO> Lightstreamer Server starting in ENTERPRISE edition.
23-gen-19 15:52:21,054 < INFO> Pump pool size set by default at 4.
23-gen-19 15:52:22,004 < INFO> Events pool size set by default at 4.
23-gen-19 15:52:22,076 < INFO> Snapshot pool size set by default at 10.
23-gen-19 15:52:22,702 < INFO> Loading Data Adapter PROXY_HELLOWORLD_SOCKETS.DEFAULT
23-gen-19 15:52:22,703 < INFO> Loading Metadata Adapter for Adapter Set PROXY_HELLOWORLD_SOCKETS
23-gen-19 15:52:22,763 < INFO> Finished loading Metadata Adapter PROXY_HELLOWORLD_SOCKETS
23-gen-19 15:52:23,704 < INFO> Connecting...
23-gen-19 15:52:23,817 < INFO> Waiting for a connection on port 7002...
23-gen-19 15:52:23,819 < INFO> Waiting for a connection on port 7001...
23-gen-19 15:52:25,957 < INFO> Connected on port 7001 from /127.0.0.1:50405
23-gen-19 15:52:26,618 < INFO> Connected on port 7002 from /127.0.0.1:50406
23-gen-19 15:52:26,620 < INFO> Connected
23-gen-19 15:52:26,823 < INFO> Request sender 'DEMO.QUOTE_ADAPTER' starting...
23-gen-19 15:52:26,865 < INFO> Reply receiver 'DEMO.QUOTE_ADAPTER' starting...
23-gen-19 15:52:26,939 < INFO> Notify receiver 'DEMO.QUOTE_ADAPTER' starting...
23-gen-19 15:52:26,987 < INFO> Finished loading Data Adapter PROXY_HELLOWORLD_SOCKETS.DEFAULT
23-gen-19 15:52:27,145 < INFO> Selector maximum load set by default at 0.
23-gen-19 15:52:27,285 < INFO> Created selector thread: NIO WRITE SELECTOR 1.
23-gen-19 15:52:27,323 < INFO> Created selector thread: NIO READ SELECTOR 1.
23-gen-19 15:52:27,327 < INFO> Created selector thread: NIO CHECK SELECTOR 1.
23-gen-19 15:52:27,406 < INFO> Lightstreamer Server initialized.
23-gen-19 15:52:27,407 < INFO> Lightstreamer Server 7.1.0 b4 build 1913 starting...
23-gen-19 15:52:28,078 < INFO> Server "Lightstreamer HTTP Server" listening to *:8080 ....
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

### Available improvements

#### Add Encryption

Each TCP connection from a Remote Adapter can be encrypted via TLS. To have the Proxy Data Adapter accept only TLS connections, a suitable configuration should be added in adapters.xml in the <data_provider> block, like this:
```xml
  <data_provider>
    ...
    <param name="tls">Y</param>
    <param name="tls.keystore.type">JKS</param>
    <param name="tls.keystore.keystore_file">./myserver.keystore</param>
    <param name="tls.keystore.keystore_password.type">text</param>
    <param name="tls.keystore.keystore_password">xxxxxxxxxx</param>
    ...
  </data_provider>
```
This requires that a suitable keystore with a valid certificate is provided. See the configuration details in the [provided template](https://lightstreamer.com/docs/ls-ARI/latest/adapter_conf_template/adapters.xml).
NOTE: For your experiments, you can configure the adapters.xml to use the same JKS keystore "myserver.keystore" provided out of the box in the Lightstreamer distribution. Since this keystore contains an invalid certificate, remember to configure your local environment to "trust" it.
The same settings will apply to both connections: "request/response" and "asynchronous".
The above example can be easily ported to the new configuration. You will just need to use a tool that supports TLS connections instead of telnet. For instance you can use openssl with the "s_client" command. Note that the same hostname supported by the provided certificate must be used for the connection.

The same settings are available for the Remote Metadata Adapter case.

#### Add Authentication

Each TCP connection from a Remote Adapter can be subject to Remote Adapter authentication through the submission of user/password credentials. To enforce credential check on the Proxy Data Adapter, a suitable configuration should be added in adapters.xml in the <data_provider> block, like this:
```xml
  <data_provider>
    ...
    <param name="auth">Y</param>
    <param name="auth.credentials.1.user">user1</param>
    <param name="auth.credentials.1.password">pwd1</param>
    ...
  </data_provider>
```
See the configuration details in the [provided template](https://lightstreamer.com/docs/ls-ARI/latest/adapter_conf_template/adapters.xml)
The same settings will apply to both connections: "request/response" and "asynchronous".
The above example can be easily extended to the new configuration. There are only two differences:

* The response to the initialization request received in the request/response window should be like this:
```cmd
  10000014209a460b9|DPI|S|ARI.version|S|1.8.2|S|user|S|user1|S|password|S|pwd1
```
*Note: Replace "10000014209a460b9" with the actual ID you received, otherwise the initialization will not succeed and you will see a warning in the log window.*
* At the same time, a new message should be issued on the asynchronous window, that should be like this:
```cmd
  0|DPNI|S|ARI.version|S|1.8.2|S|user|S|user1|S|password|S|pwd1
```

The same settings are available for the Remote Metadata Adapter case.

Authentication can (and should) be combined with TLS encryption.

## See Also

### Clients Using This Adapter

<!-- START RELATED_ENTRIES -->

* [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript)

<!-- END RELATED_ENTRIES -->

### Related Projects

* [Complete list of "Hello World" Adapter implementations with other technologies](https://github.com/Lightstreamer?utf8=%E2%9C%93&q=Lightstreamer-example-HelloWorld-adapter&type=&language=)
* [Lightstreamer - Reusable Metadata Adapters - Java Adapter](https://github.com/Lightstreamer/Lightstreamer-example-ReusableMetadata-adapter-java)

## Lightstreamer Compatibility Notes

- Compatible with Lightstreamer SDK for Generic Adapters version 1.8.2 or newer (Corresponding to Server version 7.1 or newer).
- Compatible with Lightstreamer JavaScript Client Library version 6.0 or newer.
- For a version of this example compatible with SDK for Generic Adapters 1.7 to 1.8.0, please refer to [this tag](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket/tree/for_Lightstreamer_7_0).
- For a version of this example compatible with SDK for Generic Adapters 1.4.3, please refer to [this tag](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-socket/tree/for_Lightstreamer_5.1).

## Final Notes
For more information, please [visit our website](http://www.lightstreamer.com/) and [post to our support forums](http://forums.lightstreamer.com) any feedback or questions you might have. Thanks!
