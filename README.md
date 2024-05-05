# Trace Event Logger

This logger is based on JUL to allow fast JSON traces to be written to disk. It is not lockless or nanosecond precise, but is fast and simple to use and configure.

A Snapshot taker is provided in order to write slow transactions onto disk.

An asynchronous writer is available for putting the IO bottleneck on another thread. It is typically much faster than the FileHandler from JUL.

This is a logger helper designed to facilitate entry-exit analysis. The events generated by this logger are saved in a JSON-like message format, providing extra information associated with each event. The typical event types include:

- **Durations:**
  - **B:** Begin
  - **E:** End
  - **X:** Complete (event with a duration field)
  - **i:** Instant / Info

- **Asynchronous Nested Messages:**
  - **b:** Nested Begin
  - **n:** Nested Info
  - **e:** Nested End

- **Flows:**
  - **s:** Flow Begin
  - **t:** Flow Step (Info)
  - **f:** Flow End

- **Object Tracking:**
  - **N:** Object Created
  - **D:** Object Destroyed

- **Mark Events:**
  - **R:** Marker Event

- **Counter Events:**
  - **C:** Counter Event

## Usage

To use **durations** and/or **flows**, refer to ScopeLog and FlowScopeLog. Durations are typically used to instrument simple methods, while flows are preferred when there are links to be made with other threads.

To use **Asynchronous Nested Messages**, check traceAsyncStart and traceAsyncEnd].

To use **Object Tracking**, see traceObjectCreation and traceObjectDestruction.

To use free form tracing, check traceInstant.

### Examples


To instrument

```java
import java.io.FileWriter;

public class SlappyWag {
    public static void main(String[] args) {
        System.out.println("The program will write hello 10x between two scope logs\n");
        try (FileWriter fw = new FileWriter("test.txt")) {
            for (int i = 0; i < 10; i++) {
                fw.write("Hello world "+ i);
            }
        }
    }
}
```

one needs to add to the try-with-resources block a scopewriter

```java
import java.io.FileWriter;
import java.util.logging.Level;
import java.util.logging.Logger;
import org.eclipse.tracecompass.trace_event_logger.LogUtils;

public class SlappyWag {
    
    private static Logger logger = Logger.getAnonymousLogger();
    
    public static void main(String[] args) {
        // Set the logger level based on your requirements
        // logger.setLevel(Level.FINE);

        System.out.println("The program will write hello 10x between two scope logs\n");
        try (LogUtils.ScopeLog sl = new LogUtils.ScopeLog(logger, Level.FINE, "writing to file"); FileWriter fw = new FileWriter("test.txt")) {
            for (int i = 0; i < 10; i++) {
                fw.write("Hello world "+ i);
            }
        }
    }
}
```
Will generate the following trace

```json
{"ts":0,"ph":"B","tid":1,"name":"writing to file"}
{"ts":100000,"ph":"E","tid":1}
```

example 2:

```java
try (LogUtils.ScopeLog linksLogger = new LogUtils.ScopeLog(logger, Level.CONFIG, "Perform Query")) { //$NON-NLS-1$
  ss.updateAllReferences();
  dataStore.addAll(ss.query(ts, trace));
}
```

will generate the following trace

```json
{"ts":12345,"ph":"B","tid":1,"name":"Perform Query"}
{"ts":12366,"ph":"E","tid":1}
```
See more examples in javadoc.

**P.S.: You may need to terminate the logger thread at the end of your program since the library cannot terminate it automatically.**
```Java
import java.util.logging.LogManager;
import java.util.logging.Logger;

public class SlappyWag {

  private static Logger logger = Logger.getAnonymousLogger();

  public static void main (String[] args) {
    // Your code
    // ...
    
    LogManager.getLogManager().reset();

    // End of your program
  }

}
```

## Viewing results

While one could open the traces in their favorite text editor, results are better with a GUI. One could open the resulting json files in either `chrome://tracing` or [Eclipse Trace Compass](https://eclipse.dev/tracecompass/). You will need to install trace event parser support by clicking on the `Tools->Add-ons...` menu and selecting **Trace Compass TraceEvent Parser** in trace compass to load the JSON files. If the trace is malformed due to a handler not being configured properly, the program `jsonify.py` supplied in the root of the project can help restore it. 

```console
// python 3 jsonify.py -input -output
python3 jsonify.py LOG_FILE log.json
```

[Video tutorial](https://www.youtube.com/watch?v=YCdzmcpOrK4)

## Performance

On an Intel i5-1145G7 @ 2.60GHz with an NVME hard drive, using an `AsyncFileHandler` instead of the classic `FileHandler` leads to events being logged from 45 us/event (`FileHandler`) to 1.1 us/event (`AsyncFileHandler`). In other words, `AsyncFileHandler` can write 900k events in the time it takes FileHandler to write 22k events
One could also take advantage of the cache effect. If the data is not saturating the IO, speed is even higher.

## Design Philosophy

The design philosophy of this class is heavily inspired by the trace event format of Google. The full specification is available [here](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/edit?pli=1#).

The main goals of this logger helper are clarity of output and simplicity for the developer. While performance is a nice-to-have, it is not the main concern of this helper. A minor performance impact compared to simply logging the events is to be expected.

