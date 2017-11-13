# Websockets with Play Framework, Scala.js, Binding.scala

This project is based on:
* [Algomancer's Starter Kit](https://github.com/Algomancer/Full-Stack-Scala-Starter)
* [Lightbend's Websocket example](https://github.com/playframework/play-scala-websocket-example)

See the general setup on the starter kit page.

This is an example application showing how you can integrate a Play project with a Scala.js, Binding.scala 
project - using **Web Sockets**.

## Business Case

It's about an automatic process that can be started manually (button). 
But it should run only once at a time.

So only one actor (AdapterActor) will run the process and make sure, 
that the process is only run once at a time.

The web-sockets are created according to the 
[Lightbend's Websocket example](https://github.com/playframework/play-scala-websocket-example).

Each client sees the LogEntries of the last 'Adapter process' (LogReport) - 
or if the process is running - each LogEntry right away.

## Shared
The great thing about a **full-stack Scala app** is that 
we only have to define our domain model once for the server and the client.

Next to the model all that is needed is the JSON-un-/-marshalling. 
Thanks to the [Play JSON Derived Codecs](https://github.com/julienrf/play-json-derived-codecs) this involves only a few lines of code.

We define some messages for the communication:

```scala
// trait for all messages
sealed trait AdapterMsg

object AdapterMsg {
  // marshalling and unmarshalling
  // with json.validate[AdapterMsg] or Json.parse(adapterMsg)
  // this line is enough with this library - as AdapterMsg is a sealed trait
  // be aware that if you want for example json.validate[RunAdapter] you also need a OFormat[RunAdapter]
  implicit val jsonFormat: OFormat[AdapterMsg] = derived.oformat[AdapterMsg]()
}

// a client want's to start the Adapter process
case class RunAdapter(userName: String = "Anonymous") extends AdapterMsg

// the server indicates that the Adapter process is already running
// logReport: the LogReport of the active run.
case class AdapterRunning(logReport: LogReport) extends AdapterMsg

// the server indicates that the Adapter process is NOT running
// logReport: the LogReport of the last run - if there is one.
case class AdapterNotRunning(logReport: Option[LogReport]) extends AdapterMsg

// each LogEntry that is created by the AdapterProcess
case class LogEntryMsg(logEntry: LogEntry) extends AdapterMsg

// sent when the Adapter Process finished
case class RunFinished(logReport: LogReport) extends AdapterMsg

// as with akka-http the web-socket connection will be closed when idle for too long.
case object KeepAliveMsg extends AdapterMsg

```

## Client
The UI is done with 
[Binding.scala](https://github.com/ThoughtWorksInc/Binding.scala)

It is more or less HTML-snippets that contain dynamic content provided by Binding.scala:
* `Vars[LogEntry]` a list of LogEntries that from the active- or last Adapter run.
* `Var[Boolean]` is true if the Adapter process is running at the moment.
* `Var[Option[LogLevel]]` the LogLevel of the last Adapter run - used for the title.

If you have troubles understanding it, please check out [Binding.scala-Google-Maps](https://github.com/pme123/Binding.scala-Google-Maps), where I explained all the details.

## Server
When you go to [http://localhost:9000](http://localhost:9000) 
a web-socket is opened to show you the active- or last LogReport of the Adapter Process.

### HomeController
The web-sockets are managed with Akka Actors. The implementation was taken from 
the **[Lightbend's Websocket example](https://github.com/playframework/play-scala-websocket-example)** 
and adjusted to fit my needs.

Only difference, I made the allowed hosts configurable:
```scala
  def originMatches(origin: String): Boolean = {
    import scala.collection.JavaConverters._

    val allowedHosts = config.underlying.getStringList("wsocket.hosts.allowed").asScala
    allowedHosts.exists(origin.endsWith)
  }
```

### UserParentActor
This Actor exists only once and is a factory to create the web-socket for a web-client.
### UserActor
The Actor that manages the web-socket: 

* When create it subscribes itself to the AdapterActor.
* When stopped it un-subscribes itself from the AdapterActor.

### AdapterActor
This Actor handles the business logic.

```scala
    // called if a client runs the Adapter Process (Button)
    case RunAdapter(user) =>
      if (isRunning) // this should not happen as the button is disabled, if running
        log.warning("The adapter is running already!")
      else {
        log.info(s"run Adapter: $sender")
        runAdapter(user) // the dummy logic that runs the process
      }
```
Each AdapterMsg is send to all UserActors:
```scala
  private def sendToSubscriber(adapterMsg: AdapterMsg): Unit =
    userActors.values
      .foreach(_ ! adapterMsg)
```

# Run the application
```shell
$ sbt
> run
```
open [http://localhost:9000](http://localhost:9000) in a browser.
