# Chapter 2 Concepts, architecture, and deployment of Fluentd 
### 2.2.6 Deploying a log generator
Ideally, we want to prove our configuration for input plugins and confirm configuration for things like log rotation. We want a configuration-driven utility that can continuously send log events. We have one available at https://github.com/mp3monster/LogGenerator and will be using this in the subsequent chapters. This tool provides several helpful features for us:

* Take an existing log file, replay the log events from an existing log file, write them with current timestamps, and write the logs with the same time intervals as the logs had when originally written.
* Take a test log file that describes the time gap and log body, and play it back with the correct intervals between events.
* Write log files based on a pattern, meaning different log formats can be generated.
* Send the logs via the Java logging framework to simulate an application using a logging framework.

The simulator uses a properties file to control its behavior and uses a file thatdescribes a series of log entries to replay. We will use this in later chapters to see how log rotation and other behaviors can work. Each book chapter has a folder containing  the relevant properties files and log sources to help with that chapter, as shown in figure 2.5. With the LogSimulator copied into the download root folder as previously recommended, run this command:

```bash
groovy LogSimulator.groovy Chapter2\\SimulatorConfig\\tool.properties
```
#### RUNNING LOGSIMULATOR AS A JAR
To use the JAR version of the LogSimulator, the JAR file needs to be downloaded into the parent directory of all the chapter resource folders. Then the command can have the Groovy LogSimulator.groovy element replaced with java -jar LogSimulator.jar, so the command would appear as

```bash
java -jar LogSimulator.jar Chapter2\\SimulatorConfig\\tool.properties
```
#### LOGSIMULATOR IN MORE DETAIL
If you would like to know what is going on in more depth, then edit the tool.properties file and change the ``verbose`` property from ``false`` to ``true``. This will display to the console log entries that are defined in the file small-source.txt. All the properties for the simulator are explained in the documentation at https://github.com/mp3monster/LogGenerator.

## 2.3 Bringing Fluentd to life with “Hello World”
Now that we’ve looked at the architecture of Fluentd and deployed it into an environment,let’s bring this to life.
### 2.3.1 “Hello World” scenario
The “Hello World” scenario is very simple. We will use the fact that Fluentd can receive log events through HTTP and simply see the console record the events. To start with, we will push the HTTP events using Postman. The next step will be to extend this slightly to send log events using the LogSimulator.

### 2.3.3 Starting Fluentd
As the Fluentd service is in our PATH, we can launch the process with the command fluentd anywhere. However, the tool will look in different places without a parameter defining the config location, depending on the environment and installation process. For Windows and Linux, Fluentd will try to resolve the location ``/etc/fluent/fluent.conf``. For Windows, this will fail unless the command is run within a Linux subsystem. We are not using the default to start Fluentd. We need to navigate the shell to wherever you have downloaded the configuration file or include the full path to the configuration file as the parameter. Then run the following command:
```bash
fluentd -c HelloWorld.conf
```
To run the Fluentd command from the root of the downloaded resources, which will be the norm for the rest of the book, the command would be
```bash
fluentd -c ./Chapter2/Fluentd/HelloWorld.conf
```
A new shell window is required to run the LogSimulator. Within the shell, you will need to navigate to where the configurations have been downloaded. Within each of the chapter’s folders is a folder called SimulatorConfig. Depending upon the chapter,you will find one or more property files. Inside the property file, you’ll find a series of key-value pairs that will control the LogSimulator’s behavior. This includes referencing the log file to replay or test data. These references are relative, meaning we need to be in the correct folder—the parent folder to the chapters—to start the simulator successfully. We can then start the LogSimulator with the command 
```bash
groovy LogSimulator.groovy Chapter2\SimulatorConfig\HelloWorld.properties
```
or, if you choose to use the JAR file
```bash
java -jar LogSimulator.jar Chapter2\SimulatorConfig\HelloWorld.properties
```
Remember to correct the slashes in the file path for Linux environments. The Log-Simulator is provided with a configuration that will send log events using a log file source using the same HTTP endpoint. This will result in each of the log events being displayed on the console.
## 2.4 “Hello World” with Fluent Bit
downloading one of the prebuilt binaries or using one of the supported package managers, such as apt and yum. For Windows, Treasure Data has provided Windows binaries (available at https://docs.fluentbit.io/manual/installation/windows). Because the binaries are provided by Treasure Data, the created artifacts make use of the prefix td. For simplicity and alignment to the basic version of Fluent Bit, we recommend downloading the zip version. We have used the zip download approach for our examples.

Unpack the zip file to a suitable location (we will assume ``C:\td-agent``) as the location. To make life easier, it is worth adding the bin folder (e.g., ``C:\tdagent\bin``) into the PATH environmental variables, as we did with Fluentd. 

We can check that Fluent Bit has been deployed with the following simple command:
```bash
fluent-bit -–help
```
This will prompt Fluent Bit to display its help information on the console.
### 2.4.1 Starting Fluent Bit
The obvious assumption would be that as long as we limit our Fluentd configuration file to the plugins available in a Fluent Bit deployment, we can use the same configuration file. Unfortunately not—while the configuration files are similar, they aren’t the same.

We’ll explore the difference in a while. But to get Fluent Bit running with our “Hello World” example, let’s start things with a configuration file previously prepared, using the command
```bash
fluent-bit -c ./Chapter2/FluentBit/HelloWorld.conf
```
As a result, Fluent Bit will start up with the configuration provided. Unlike Fluentd, Fluent Bit’s support for HTTP is more recent and may not have all the features you want, depending on when you read this. Therefore, it is possible to match Fluentd for HTTP in our scenario of sending JSON. If you bump up against HTTP feature restrictions, then you can at least drop down to using the TCP plugin (HTTP is a layer over the TCP protocols). Both Fluent Bit and Fluentd support HTTP operations for capturing status information and HTTP forwarding. The only downside of working at the TCP layer is that we can’t use Postman to send the calls. You can create the same effect with other tools that know how to send text content to TCP sockets. For Linux, utilities such as tc can do this. In a Windows environment, there isn’t the same native tooling. It is possible to create a Telnet session using tools such as PuTTY (www.putty.org), and LogSimulator includes the ability to send text log events to a TCP port. For Fluent Bit, let’s use Postman for HTTP and use the LogSimulator for TCP. Starting with TCP, the following command will start the LogSimulator, providing it with a properties file and a file of log events to send. As we have already installed this tool, we can start it up. Using a separate shell (with the correct Java and Groovy versions), we can run the command
```bash
groovy LogSimulator.groovy Chapter2\SimulatorConfig\fb-HelloWorld.properties   .\TestData\small-source.json
```

You may have noticed a lag between the simulator starting and seeing Fluent Bit displaying events. This reflects that one of the configuration options is the time interval when the cache of received log messages is flushed to the output. As we will discover later in the book, this is one of the areas that we can tune to help performance.

#### NOW WITH HTTP
The difference between the TCP and HTTP configurations is small, so you can either make the changes to the ``Chapter2/FluentBit/HelloWorld.conf`` or use the provided configuration file ``Chapter2/FluentBit/HelloWorld-HTTP.conf``. The following shows the changes that need to be applied:

* In the Input section, change the Name tcp to Name http.
* As we have been using port 18080 for HTTP in Postman, let’s correct the port in the configuration, replacing port 28080 with port 18080.

Save these changes once applied. To see how Fluent Bit will work now, stop the current Fluent Bit process if it’s still running. Then restart as before, or using the provided changes, start with
```bash
fluent-bit -c ./Chapter2/FluentBit/HelloWorld-HTTP.conf
```
Once running, use the same Postman settings to send the events as we did for Fluentd.

### 2.4.2 Alternate Fluent Bit startup options

Fluent Bit can also be configured entirely through the command line. This makes an effective way to configure Fluent Bit, as it simplifies the deployment (no mapping of configuration files needed). However, this does come at the price of readability. For example, we could repeat the same configuration of Fluent Bit with

```bash
fluent-bit -i tcp://0.0.0.0:28080 -o stdout
```
If you run this command with the simulator as previously set up, the outcomes will be the same as before. Fluent Bit, like Fluentd, isn’t tied to working with a single source of log events. We can illustrate this by adding additional input definitions into the command line. While running in a Windows environment, let’s add the ``winlog`` events to our inputs. For Linux users, you could replace the ``winlog`` source with ``cpu`` and ask Fluent Bit to tell us a bit more about what it is doing by repeating the same exercise, but with the command

```bash
fluent-bit -i tcp://0.0.0.0:28080 -i winlog -o stdout -vv
```

This time we will see several differences. First, when Fluent Bit starts up, it will give us a lot more information, including clearly showing the inputs and outputs being handled. This results from the ``-vv`` (more on this in the next section). As the log events occur, in addition to our log simulator events, the ``winlog`` information will be interleaved.

#### FLUENTD AND FLUENT BIT INTERNAL LOGGING LEVELS
Both Fluentd and Fluent Bit support the same command-line parameters that can control how much information they log about their activities (as opposed to any log-level information associated with a log event received). In addition to being controlled by the command line, this configuration can be set via the configuration file. Both tools recognize five levels of logs, and when no parameter or configuration is applied, the midlevel (info) is used as the default log level. Table 2.5 shows the log levels, the command-line parameters, and the equivalent configuration setting. The easiest
way to remember the command line is -v is for ``verbose`` and -q is for ``quiet``; more letters increase verbosity or quietness.

Table 2.5 Log levels recognized by Fluentd and Fluent Bit
| Log level | Command line  | Configuration setting |
|-----------|---------------|-----------------------|
| Trace     | -vv           | Log_Level trace       |
| Debug     | -v            | Log_Level debug       |
| Info      |               | Log_Level info        |
| Warning   | -q            | Log_Level warn        |
| Error     | -qq           | Log_Level error       |
> **⚠ NOTE:** 
> Trace level setting will occur only if Fluent Bit has been compiled with the ``build flag`` set to enable trace. This can be checked using the Fluent Bit help command (``fluent-bit -h`` or ``fluent-bit -–help``) to display a list of the build flags and their settings. Trace-level logging should be needed only while developing a plugin.

### 2.4.3 Fluent Bit configuration file comparison
Previously we mentioned that the Fluentd and Fluent Bit configurations differ. To help illustrate the differences, table 2.6 offers the configuration side by side.

Table 2.6 Fluentd and Fluent Bit configuration comparison (using the HTTP configuration of Fluent Bit)

<table>
<tr>
<td> Fluent Bit </td> <td> Fluentd  </td>
</tr>
<tr>
<td>

```properties
# Hello World configuration will take events received
# on port 18080 using TCP as a protocol
[SERVICE]
   Flush 1
   Daemon Off
   Log_Level info
# define the TCP source which will provide log events
[INPUT]
   Name http
   Host 0.0.0.0
   Port 18080
# accept all log events regardless of tag and write
# them to the console
[OUTPUT]
   Name stdout
   Match *
```
</td>
<td>

```properties
# Hello World configuration will take events received on port 18080 using
# HTTP as a protocol
# set Fluentd's configuration parameters
<system>
   Log_Level info
</system>
# define the HTTP source which will provide log events
<source>
   @type http
   port 18080
</source> # after a directive
# accept all log events regardless of tag and write them to the console
<match *>
   @type stdout
</match>
```

</td>
</tr>
</table>

If you want to play spot the difference, then you should have observed the following:
* Rather than directives being defined by opening and closing angle brackets``(<>)``, the directive is in square brackets ``([])``, and the termination is implicit by the following directive or end of the file.
* ``SERVICE`` replaces the system for defining the general configuration.
* ``@type`` is replaced by the ``Name`` attribute to define the plugin to be used.
* ``Match``, rather than being the name of the directive with a parameter in the directive, becomes ``Output``. The ``match`` clause is then defined by another namevalue pair in the attributes.
* Older versions of Fluent Bit didn’t support HTTP, so events would need to be sent using events using TCP, but the events received can still be in JSON format.

When looking at the configurations side by side, the details aren’t too radically different,but they are significant enough to catch people out.

 

