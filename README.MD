# Eventbus-Bridge

This is an implementation of the event bus bridge that allows Java applications to send and receive event bus messages using the SockJS websocket sub-protocol.
The API mimics the one used in `vertxbus.js`.
Internally, vert.x is used for websocket communication to the SockJS service that governs which event bus addresses are available.

Why would I use this? You can use this in cases you don't want to join a Vert.x cluster for event bus communication.

## Usage

EventBusBridge is the main class to use. Start with the connect method and take it from there.

### Simple Example

Call the static connect method with an URI and a connection handler.
Since connecting to the event bus is asynchronous, the EventBusBridge instance to use for registering message handlers and sending messages `eb` is being passed
in as parameter for the connection handler and is ready for registering handlers and sending messages.
You should not be using the EventBusBridge instance returned by connect before `isOpen` returns true which is why it is recommended to only interact with 
it in the connection handler as seen below:


```java
EventBusBridge.connect(URI.create("http://localhost:8765/bridge"), eb -> {
    eb.registerHandler("test", msg -> {
         System.out.println("I gots a message:" + msg);
     });
     eb.publish("test", "hello");
});
```

For this example to work, a SockJS service needs to run that allows incoming and outgoing messages on the `test` address.
Here's an example:

```java
vertx = Vertx.vertx();
Router router = Router.router(vertx);

// events specific to THOPs are made available over the bridge
SockJSHandler sockJSHandler = SockJSHandler.create(vertx);
BridgeOptions options = new BridgeOptions();
options.addOutboundPermitted(new PermittedOptions().setAddress("test"));
options.addInboundPermitted(new PermittedOptions().setAddress("test"));
sockJSHandler.bridge(options);

router.route("/bridge/*").handler(sockJSHandler);
vertx.createHttpServer().requestHandler(router::accept).listen(8765, (res) -> {
    System.out.println("I'm here to serve");
});
```

Registering handlers comes in two flavors `EventHandler` and `MessageHandler`.
This boils down to you having to store the reference to the EventBusBridge somewhere or it being passed back to you.
Here's an example that shows the difference:

```java
eb.registerHandler("test", msg -> { ... } );
eb.registerHandler("test", (msg, eventBus) -> { ... });
```

### Message payload

While Vert.x supports arbitrary message types on the event bus, this bridge only supports plain text and JsonObjects as message payload.
Accessing the payload as JsonObject is a bit cumbersome as the type of the message parameter must be specified:

```java
eb.registerHandler("test", (EventBusBridge.EventBusMessage<JsonObject> msg) -> {
    assertEquals("world", msg.body().getString("hello"));
});
eb.publish("test", new JsonObject().put("hello", "world"));

```

For convenience, `msg.asJson()` will return an `Message<JsonObject>` so the code above can be simplified like this:

```java
eb.registerHandler("test", msg -> {
    assertEquals("world", msg.asJson().body().getString("hello"));
});
eb.publish("test", new JsonObject().put("hello", "world"));

```


### Registering/Unregistering handlers

There are some gotchas when using lambda expressions and unregistering handlers for messages.
For example, this will fail:

```java
eb.registerHandler("test", this::myHandler):
eb.unregisterHandler("test", this::myHandler);
```

since the compiler will create different lambda classes for each occurrence of `this::myHandler`!

In order to make the most common de-registration case simple (de-register after receiving a message),
the message object has an unregister method that will unregister the current handler.

```java
eb.registerHandler("test", msg -> { System.out.println("Got your message. Now leave me alone!"); msg.unregister(); }); 

```
Note one important caveat which is that `msg.unregister` will only work correctly if used while the handler is being called.


## Status

0.1 release, not particularly well-tested. Use at your own risk.

## Installation

Since it is early days, download the git project and run `mvn jar:jar` to get something you can add to your Java project.


## Feedback

Open a ticket or send an email to beders (at) saffrontech (dot) com