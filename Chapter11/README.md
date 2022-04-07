## 11.6 Fluentd’s own logging and appenders
Table 11.1 Where Fluentd can integrate into a native or commonly used framework directly or indirectly
| Language | Has native logging library | Fluentd logger library                         | Alternate open source solution                                      |
|----------|----------------------------|------------------------------------------------|---------------------------------------------------------------------|
| Erlang   | Y                          | https://github.com/fluent/fluent-logger-erlang |                                                                     |
| Go       | N                          | https://github.com/fluent/fluent-logger-golang |                                                                     |
| Java     | N                          | https://github.com/fluent/fluent-logger-java   | Log4J: https://github.com/tuxetuxe/fluentd4log4j                    |
| Node.js  | N                          | https://github.com/fluent/fluent-logger-node   | Directly integrates with Log4JS                                     |
| OCaml    | N                          | https://github.com/fluent/fluent-logger-ocaml  |                                                                     |
| Perl     | N                          | https://github.com/fluent/fluent-logger-perl   | Log4perl: https://metacpan.org/pod/Log::Log4perl::Appender::Fluent |
| PHP      | N                          | https://github.com/fluent/fluent-logger-php    | https://github.com/Seldaek/monolog                                  |
| Python   | Y                          | https://github.com/fluent/fluent-logger-python |                                                                     |
| Ruby     | Y                          | https://github.com/fluent/fluent-logger-ruby   |                                                                     |
| Scala    | N                          | https://github.com/fluent/fluent-logger-scala  | Via Logback compatibility with SLF4S                                |

If you don’t have a suitable logging framework or wrapper layer, then there is the
option to use the Fluentd logging library directly within your core application code.
As with all things, there are pros and cons to such an approach. To that end, in table
11.2, we’ve called out the pros and cons of using the libraries directly to help you
make informed decisions.


Table 11.2 Pros and cons of using Fluentd’s own log framework
<table>
    <tr><td>Pros</td><td>Cons</td></tr>
    <tr><td>Small footprint, as it is only providing for output to Fluentd.</td><td>Locked into using Fluentd. For packaged solutions,you had better not try to force its logging to work differently from the options it provides. This is where it may be better to consider a custom plugin or find a compromise configuration.</td></tr>
    <tr><td>The Fluentd logging library provides the same programmer interface as other frameworks, giving a comparable development experience. But the Fluentd server offers significantly more sophistication than a logging framework for handling the log events.</td><td></td></tr>
    <tr><td>Communication to Fluentd is over the network.Using msgpack compression means efficient communication and can limit hosting complexities (e.g., external storage for containers, complexities of storage for FaaS).</td><td></td></tr>
    <tr><td></td><td></td></tr>
</table>
The final possible option for logging directly to Fluentd is to leverage a framework’s
plugins to communicate using TCP/IP or HTTP(s) and send log events using those
protocols. These routes mean you have no library dependencies (assuming your programming
language can provide basic networking).

### 11.7.1 Python with logging framework: Using the Fluentd library
In most situations, having the Fluentd library plugging directly into the logging
framework is ideal, as we can configure different ways to log without any code
changes. Let’s start with the code that creates the logging framework driven by the
configuration file; our application then uses the framework to record a log event. To 
achieve this illustration, we need to establish some code and configuration, which we
will review in the coming listings:

* A simple Python test application to use the logging framework and generate a
log event
* A configuration file telling the logging framework how and what to log
* A Fluentd server and configuration so it can receive the log events

In the code shown in listing 11.1, we can see the Python test application, which creates
a configuration object from the configuration file and passes this into the logging
context, and then requests a logger object. With the logging object ready, we could
use that object as many times as we like. In our example, we then construct content in
the log message—here, the date-time string representation. Then the logging framework
is called twice, once as a text message and again as a JSON construct. When you
review the code, note the complete absence of Fluentd references. This is all handled
in the logging framework for us based on the configuration.

> Listing 11.1 Chapter11/clients/log-conf.py—Test Python client—configuration only
```py
import logging
import logging.config
import yaml
import datetime

with open('logging.yaml') as fd:           # Loads the configuration file, which will describe the logging setup wanted
    conf = yaml.load(fd, Loader=yaml.FullLoader)

logging.config.dictConfig(conf['logging'])

log = logging.getLogger()                  # Gets the correct logger object; by not providing a specific name, we will be given the default setting 
now = datetime.datetime.now().strftime("%d-%m-%Y %H-%M-%S")

log.warning ('from log-conf.py at ' + now) # Generates log event

log.info ('{"from": "log-conf.py", "now": '+now+'"}') # It would be preferable to create JSON for the log event being passed to the library.
                                                      # But building JSON objects first distracts from the point we’re trying to make.
```
The configuration shown in listing 11.2 is the detail loaded in the application and
interpreted by the logging framework to establish the desired ways of logging. Only
the configuration will drive logging to communicate to Fluentd directly through the
definition of **handlers** (or appenders, using the naming we described earlier). Note
how the configuration entities relate back to the structure illustrated in figure 11.1,
with loggers referencing a handler (appender) by name and the handler referencing
a formatter implemented to work with Fluentd. Here the filters are not decoupled but
specified within the loggers and handlers using the ``level`` attribute.

> Listing 11.2 Chapter11/clients/logging.yaml—Test Python client: configuration
```yaml
logging:
  version: 1

  formatters:      # Defines how the log event will be represented
    fluent_fmt:
      '()': fluent.handler.FluentRecordFormatter   # We still need to tell the logging framework which class will implement the formatter interface. 
                                                   # Here we have a customer formatter for Fluentd, so the server will receive a correctly structured event.
      format:
        level: '%(levelname)s'
        hostname: '%(hostname)s'
        where: '%(module)s.%(funcName)s'

  handlers:     # Defines the handler (the appender) and provides it with the configuration necessary to communicate with our Fluentd node
    fluent:
      class: fluent.handler.FluentHandler   # Identifies the class that knows how to actually communicate with the Fluentd server
      host: localhost
      tag: test
      port: 18090
      level: DEBUG
      formatter: fluent_fmt

  loggers:           # Defines the default logger object and the default settings, and links the default logging configuration to the relevant handlers
    '': # root logger
      handlers: [fluent]
      level: DEBUG
      propagate: False    
```
For Fluentd to work with the examples, we have provided a configuration file. If you
review the configuration file in listing 11.3, you’ll see two sources configured. The
source using the ``forward`` input plugin will receive the log events. You can confirm
that by comparing the port number in the configuration to the logger YAML file; we
will see the other source put to use shortly.

> Listing 11.3 Chapter11/fluentd/http.conf—simple HTTP source
```conf
<system>
    Log_Level debug # Sets the logging to debug to help us understand what is happening. 
                    # Given that HTTP is highly configurable in its behavior, let’s ensure that assumed configurations are compatible.
</system>

<source>
    @type http
    port 18080
    <parse>
        @type none # Let’s not worry about the expected structure at this stage and process the log event as a single string received over HTTP.
    </parse>
</source>

<source>
    @type http
    port 18085
    <parse>
        @type json # Using a different port, we can take the log event using HTTP in a JSON structure.
    </parse>
</source>

<source>   # Using the forward plugin, we can receive the payload as JSON text or compressed with msgpack.
    @type forward
    port 18090
</source> 


# accept all log events regardless of tag and write them to the console
<match *>
    <format>
        @type json
    </format>
    @type stdout
</match>
```
To start things up, we’ll need two shell windows; having navigated to the folder with all
the provided resources, Fluentd can be started with the following command:

```bash
fluentd -c Chapter11/Fluentd/http.conf
```

This is followed by navigating to the ``Chapter11/clients`` folder and executing the
command

```bash
python log-conf.py
```

Once executed, you will see that the Fluentd server will write the generated log event
to the console in a JSON format.

### 11.7.2 Invoking Fluentd appender directly
Let’s now look at how the code may appear using the Fluentd Python library directly
from our application instead of a logging framework. While this is the Python implementation,
the logger libraries work similarly for most of the supported languages.
Obviously, each implementation may have differences because of the constraints of
how the programming language works. For example, Go doesn’t have classes and
inheritance like Python and Java, but rather has modules and types.

To make it easy to compare the direct calls to the Fluentd logging library approach
using the logging framework, we have created a new Python test client shown in listing
11.4. The first immediate difference is the client we need to explicitly import the Fluentd
library into our code. Our code no longer establishes the logger context and logger
object but interacts with a sender, which is a specific implementation of an
appender (or handler, as Python calls it). The sender object is constructed with the
configuration needed to connect to Fluentd (you could, of course, retrieve this data
from a generic configuration file). As before, we’re constructing the time to put into
the log event message. Then, finally, we can use either the Fluent library’s ``emit`` or
``emit_with_timestamp`` functions to transmit the log event. The emit functions
require the payload to be represented as a hashmap (or dictionary, using Python’s
naming).

> Listing 11.4 Chapter11/clients/log-fluent.py—Direct Fluentd library use
```py
#This implementation makes use of the Fluentd implementation directly without the use of
# the Python logging framework
import datetime, time
from fluent import handler, sender  # The import making a direct dependency on the Fluentd library

fluentSender = sender.FluentSender('test', host='localhost', port=18090)   # Creates an instance of the Fluentd handler
# using the Fluentd Handler means that msgpack will be used and therefore the source plugin in Fluentd is a forward plugin.
now = datetime.datetime.now().strftime("%d-%m-%Y %H-%M-%S")
fluentSender.emit_with_time('', int(time.time()), {'from': 'log-fluent', 'at': now})  # Sends the log event
```
To see this scenario run, restart Fluentd as we did in section 11.7.1. This means that
the log events will, when received, be displayed in Fluentd’s console session. Then, in
the second shell, we need to run the command (from ``Chapter11/clients`` folder)

```bash
python log-fluent.py
```
### 11.7.3 Illustration with only Python’s logging
In the examples so far, the logging has used the Fluentd logging library directly (using
its sender object) and indirectly (using the Python logging framework and configuration).
This time, we’re going to look at how we can work without using the Fluentd
library at all. If you examine the code of the Fluentd library, you will find the library
uses the msgpack compression mechanism that we encountered in chapters 3 and 4.
Msgpack is part of Fluentd, not the native Python logging itself. As a result, when
working with only the native layer, we don’t benefit from the compression provided by
msgpack. The only way to overcome this would be to implement our own formatter
code that uses msgpack.

Without resorting to developing your own version of the Fluentd library, the next
option is to use a prebuilt Python logging handler (or, as we have called it, an
appender) to talk to Fluentd directly. The value of this approach for Python is nonexistent.
But it may be necessary if you wanted to use a comparative approach in another
language.

As not all languages benefit from the Fluentd library, let’s look at how things need
to be implemented without that help. In this case, we will exploit the prebuilt
``HTTPHandler`` (most languages have a comparable feature). As with the preceding
illustrations, we have provided another client implementation shown in listing 11.5.
For this to work, we instantiate a Python ``HTTPHandler`` for logging, with the necessary
connection details. Note that in the connection, we provide both the server address
and a URL path separately. Fluentd will expect a path rather than an attempt to talk to
the root address. We have provided a custom formatter and attached that to the
handler. We then go through the same formatting process to form part of the log
event and invoke the logger object with the log event string.

Using the prebuilt HTTPHandler means that the Fluentd configuration will need
an HTTP source plugin to be included, which we have already done.
> Listing 11.5 Chapter11/clients/log-simple.py—Direct Fluentd library use
```py
#This approach makes use of the Python logging without any Fluentd provided logging 
# functionality. As a result we have fallen back to sending HTTP and using a custom
# formatter to populate the values we would see through the configuration
import logging, logging.handlers    #  Needs to import the core classes for both logging and the handlers
import datetime
testHandler = logging.handlers.HTTPHandler('localhost:18080', '/test', method='POST')  #  Creates the HTTPHandler and provides the details; in a secured production environment, this would include the use of certificates as well.
custom_format = {
  'host': '%(hostname)s',
  'where': '%(module)s.%(funcName)s',
  'type': '%(levelname)s',
  'stack_trace': '%(exc_text)s'
}
myFormatter = logging.Formatter(custom_format)
testHandler.setFormatter(myFormatter)

log = logging.getLogger("test")   # Creates the logger to use if it doesn’t already exist
log.addHandler(testHandler)       # We add the log handler we created to the root log object ready for use.
now = datetime.datetime.now().strftime("%d-%m-%Y %H-%M-%S")
log.warning ('{"from":"log-simple", "at" :'+now+'"}')  # Invokes the handler. When implementing a language-specific version of this code, ideally you would use a library to  enerate the JSON rather than manually inject it.
```
To run this example, open two shells as done previously. Navigate the root folder, and
then start up Fluentd. Once running, execute the Python script in each shell using the
following commands (from the Chapter11/clients folder for the Python script):

```bash
fluentd -c Chapter11/Fluentd/http.conf
python log-simple.py
```
### 11.7.4 Illustration without Python’s logging or Fluentd library
While there is no reason to stop using the Python logging framework in the real
world, it may not be an option in other languages, so let’s see how that might look. For
continuity and ease of comparison, we’ll demonstrate what this could look like with
Python. Most languages provide the means to interact with HTTP services without any
dependencies. We can interact with the Fluentd HTTP source plugin, as we eliminated
Fluentd’s logging library. But we are now responsible for constructing all the
HTTP headers, handling the HTTP connection to keep things open, and closing the
connections, as shown in listing 11.6. This listing follows the same previous pattern of
a client file to make it easy to make side-by-side comparisons.

As you can see, code populates the header with details such as the content type
and content length. This should feel familiar, as, in many ways, these few lines of code
are exactly the same as how we configured Postman in our “Hello World” scenario in
chapter 2. As this is using the HTTP connection, we again don’t benefit from the
msgpack compression.

> Listing 11.6 Chapter11/clients/log.py—logging without any support
```py
#This implementation makes no use of the Python logging framework or the Fluentd 
# logging library. It uses pure HTTP based traffic.
import httplib, urllib
import datetime

message = '{"from":"log.py", "at":"'+datetime.datetime.now().strftime("%d-%m-%Y %H-%M-%S")+'"}'
headers = {"Content-Type": "application/JSON", "Accept": "text/plain", "Content-Length":len(message)}   # Manually populates the HTTP header attributes
conn = httplib.HTTPConnection("localhost:18085")  # Creates the connection
conn.request("POST", "/test", message, headers)   # Sends the log event

response = conn.getresponse()
print response.status, response.reason
conn.close()    # We’re responsible for closing the resources.
```
Assuming that the Fluentd server is still running from the previous examples, then all
we need to do is run the command (from the ``Chapter11/clients`` folder)

```bash
python log.py
```
As before, we should expect to see the log events being written to the Fluentd server
console.

### 11.7.5 Porting the Fluentd calls to another language into action
The company you work for is trialing some smart devices with some custom functionality
in your manufacturing facility. The trialed smart devices are currently known to
support several core languages, including Java, Python, and Ruby. The idea has been
put forward that the smart devices already call the central server when they need or
have to send data. Doing so allows battery power to be conserved by not powering
wireless until it is needed. That principle could be applied to logging any issues that
the smart devices experience. To keep the software footprint as small as possible, you
have been asked to not add any additional libraries. You have been asked to provide a
proof of concept as to whether the devices could talk directly to your current Fluentd
infrastructure rather than needing a custom solution that acts as a proxy between the
smart device and Fluentd.

#### ANSWER
Our primary languages are Java and Groovy (Groovy running on the Java Virtual
Machine). We have built a small Groovy example using native HTTP calls to a Fluentd
server. This can be found in ``Chapter11/ExerciseResults/native-fluentd.groovy``. Groovy does bring an overhead but allows us to produce the proof quickly,
as we don’t need to set up a build and package setup (and we’ve previously introduced
a Groovy setup in the book).

You should have produced a similar outcome with your preferred language and
demonstrated the result using our simple Fluentd configuration or one of your own.

### 11.7.6 Using generic appenders: The takeaways
As you have seen, working with common protocols is possible, but it does increase the
development effort. Additionally, without more effort, you do not gain the benefits of
msgpack and knowing that the library has been proven. So, if you cannot use a prebuilt
Fluentd library, consider looking for or developing a wrapper that will translate the way
the logging framework works with the interface provided by the Fluentd library.