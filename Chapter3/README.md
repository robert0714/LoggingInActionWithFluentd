### 3.3.1 HTTP interface check
Fluentd provides an HTTP endpoint that will provide information about how the instance is set up, as the following listing shows.

Listing 3.4 Chapter3/Fluentd/rotating-file-self-check.conf
```conf
<source>
    @type monitor_agent
    bind 0.0.0.0    # The address to bind to (i.e., the local server)
    port 24220      # The port to be used for this service
</source>
```
With the Fluentd running with the provided configuration (``fluentd -c Chapter3/Fluentd/rotating-file-self-check.conf``), start up Postman as we did in chapter 2. Then configure the address to be`` 0.0.0.0:24220/api/plugins.json``.As you can see in the ``bind`` attribute, as with other plugins, this relates to the DNS or IP of the host, and the ``port`` attribute matches the port part of the URL. The interface could be described as ``{bind}:{port}/api/plugins.json``. Unlike in chapter 2, where the operation was a POST, we need the operation set to be GET. Once done, click the send button, and we will see an HTTP representation of the running configuration returned, as highlighted in figure 3.1.


| URI Parameter | Description                                                                                                                                                                                       | Example              |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------|
| debug         | Will get additional plugin state information to be included in the response. The value set for the parameter does not matter.                                                                     | ?debug=0             |
| with_ivars    | The use of this parameter is sufficient for the instance_variables attribute to be included in the response. We will discuss the instance variables when we develop our plugin later in the book. | ?with_ivars=false    |
| with_config   | Overrides the default or explicit setting include_config. The value provided must be true in lowercase; all other values are treated as  alse.                                                    | ?with_config=true    |
| with_retry    | Overrides the default or explicit setting include_config. The value provided must be true or false in lowercase; all other values are treated as false.                                           | ?with_retry=true     |
| tag           | This filters the returned configuration to return only the directives linked to the tag name rovided.                                                                                            | ?tag=simpleFile      |
| @id           | This filters the response down to a specific directive. If the configuration doesn’t have an explicit ID, then the value will be arbitrary.                                                       | ?id=in_monitor_agent |
| @type         | Allows the results to be filtered by plugin type.                                                                                                                                                 | ?id=tail             |

We can also get the basic state of the plugins periodically reported within Fluentd by adding the tag and emit_interval attributes to the source directive. We can see the impact if we run the configuration with these values set using ``fluentd -c Chapter3/Fluentd/rotating-file-self-check2.conf`` (or you can try editing and adding the attributes to the previous configuration yourself). With Fluentd up and running, we will start to see some status information every 10 seconds, like the following fragment published:
```bash
2022-04-05 13:14:02.857522500 +0800 self: {"plugin_id":"RollingLog","plugin_category":"input","type":"tail","output_plugin":false,"retry_count":null,"emit_records":0,"emit_size":0,"opened_file_count":0,"closed_file_count":0,"rotated_file_count":0}
```

As the information is tagged, we can direct this traffic to a central monitoring point, all of which saves on needing to script the HTTP polling. By incorporating the source in the following listing into the configuration file (before the match), the information gets fed to the same output.

Listing 3.5 Chapter3/Fluentd/rotating-file-self-check2.conf
```conf
<source>
    @type monitor_agent
    bind 0.0.0.0
    port 24220
    @id in_monitor_agent
    include_config true   # Tells the monitor_agent to include configuration information in the output of the agent
    emit_interval 10s     # How frequently the monitoring_agent should run its self-checking and output
</source>
```

### 3.4.3 Applying a Regex parser to a complex log

As previously noted, the Regex or regular expression parser is probably the most powerful and the hardest to use. In most applications of Regex, the outcome is normally a single result, a substring, or a count of occurrences of a string. However, when it comes to Fluentd, we need the regex to produce multiple values back, such as setting the log event time, breaking down the payload to the first level of elements in the JSON event body. Let’s take a Regex expression and break it down to highlight the basics. But before we can do that, we need to work with a realistic log entry. Let’s run
up the log simulator configuration for this, using the command 
```bash
groovy LogSimulator.groovy Chapter3/SimulatorConfig/jul-log-file2.properties  ./TestData/medium-source.txt
```

Note that this is a slightly different configuration from the last example, so we can see a couple of possibilities within Fluentd. Looking at the output file generated (still ``structuredrolling-log.0.log``), the payload that is being sent now appears as

```bash
2020-04-30--20:10:52 INFO com.demo (6-1) {"log":"A clean house is the sign of a broken computer"}
```
In this output line, we can see the applied timestamp, with to-the-second accuracy,then a space followed by the log level, more space, and then a package name, followed by the numbering scheme, as previously explained. Finally, the core log entry is wrapped as a JSON structure. When we parse the message, we need to strip away that JSON notation, as it should not be in the JSON structure we want.

The goal is to end up with a structure as follows in Fluentd, so we can then use the values for future manipulation:
```json
{
    "level":"INFO",
    "class":"com.demo",
    "line":33,
    "iteration":1,
    "msg":"What is an astronauts favorite place on a computer? The Space bar!"
}
```
Here is the regular expression, with character positions added below it, to make it easy to precisely reference each piece:
```bash
(?<time>\S+)\s(?<level>[A-Z]*)\s*(?<class>\S+)[^\d]*(?<line>[\d]*)\-(?<iteration>[\d]*)\)[\s]+\{"log":"(?<msg>.*(?="\}))
123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123
0        1         2         3         4         5         6         7         8         9         10        11        12
```
As part of the expression, we need to use Regex’s ability to define groups of text. The scope of a group is defined by open and closing brackets (e.g., characters 1 and 12). To assign some source text to a JSON element, we need to use ``?<name>``, where the name is the element name to appear in the JSON. It should be the values level, class, line, iteration, and ``msg`` in our case. In addition to this, we also need to capture the log event time with the default ``time`` value. This can be seen between characters 2 and 8, for example, and between 16 and 23. Immediately after this, we can use the Regex notation to describe the text to be captured. For this, we use ``\S`` (characters 9 and 10), which means a non-whitespace character; by adding a + (character 11), we are declaring that this should happen one or more times. We will need to provide the parser with additional configuration, as we will need to declare how to
break this part of the message into a specific time.

The first character not to match the pattern is the space between the time and the log level—so we use the Regex representation for a single space (characters 13 and 14) and then start the group for the log level.

The expression defines the log level, which we know will be formed by one or more alphabetic uppercase characters. The use of the square brackets denotes a choice of values (characters 24 and 28). We could list all the possible characters within the brackets, but for readability, we’ve opted to indicate between capital A and capital Z; the hyphen between A and Z (character 27) denotes that this is a range. As the log level word will be multiple characters, we use the asterisk to note multiple. That completes the log level. So, outside the group, we need to denote the multiple whitespace characters—that is, ``\s*`` (starting at 31).

We can follow the same basic pattern used for the date and time for a path or class string. This can be seen between characters 34 and 46. To pass along the line to the following meaningful characters, we have defined a range with the Regex expression ``\d`` (characters 49 and 50), which means a numeric digit. However, by adding the circumflex ``(^)``, we negate the following value—in this case, any non-numeric character. This means that we will skip over the whitespace and the opening bracket. The groups for the line and iteration are the same—multiple digits required with a hyphen between the two groups. The hyphen is escaped (character 68) because it has meaning to Regex. We can see the same character escape for the curly bracket at character 95.

After the curly bracket starts the log detail, we know what the text is, so we can put it into the expression (starting at character 97). This underlines the importance of being exact in the expression, as non-escaped code characters will get treated as literals and therefore will be expected to be found where they occur in the Regex. If the literal character is not found as expected by the Regex, then the string being processed will be rejected.

The next new Regex trick is the use of the dot (character 112). This denotes any character; when combined with a following asterisk, then the expression becomes multiple occurrences of any character. This makes for an interesting challenge—how do we stop the closing quotes and bracket from being consumed into the msg group? This is by using a subgroup definition containing ?= (characters 115 and 116). This describes a look ahead for the following sequence when you find that sequence and then stop allocating text to the current group. As a result, the match expression is "}, but as a curly bracket has Regex meaning, we have to escape it with another slash. This does mean that any characters after this will be ignored.

In appendix B, we have included details of the Regex expressions, so you have a quick reference for building your expressions. Please note that if you research Regex elsewhere, while there is a high level of commonality in implementations, you will find subtle differences. Keep in mind that the Regex here is implemented using the Ruby language.

To complete the parser configuration, we need to tell the parser which named grouping represents the date and time (often shortened to ``date-time`` or ``date-time-group`` [DTG]) and how that date-time is represented. In our case, the date and time can be expressed using the pattern of ``%Y-%m-%d--%T``. As the time element is standard, we can use one of the short circuit formats (``%T``) described in appendix B. Finally, let’s piece all this information together and define the parser attributes (listing 3.6).

> **⚠ NOTE:** Regular expression processing is a rich and complex capability
> There are entire books devoted to the subject and many more with dedicated chapters. Here we have only scratched the surface, giving just enough to help you understand how it works within the Fluentd context. It might be worthwhile investing in a book to help. This link may also help: www.rubyguides.com/2015/06/ruby-regex/.

> **⚠ NOTE:** When the parser expression fails during processing, Fluentd will generate
a warning log entry.

Listing 3.6 Chapter3/Fluentd/rotating-file-read-regex.conf—parse extract
```conf
  <parse>
    @type regexp  # We have changed the parser type here to regexp for regular expressions.
    Expression /(?<time>\S+)\s(?<level>[A..Z]*)\s*(?<class>\S+)[^\d]*(?<line>[\d]*)\-(?<iteration>[\d]*)\)[\s]+\{"log":"(?<msg>.*(?="\}))/  # The expression needs to be provided between forward slashes. t is possible to add extra controls after the trailing slash, as we will see.
    
    time_format %Y-%m-%d--%T # time_format allows us to define the date-time format more concisely.
    time_key time # time_key is used to tell Fluentd which extracted value to use for the timestamp. By default, it will use a value called time, so technically this is redundant.     
  </parse>
```

This configuration can be run by restarting the simulator, as we did to get an example value, and then starting Fluentd with the configuration
```bash
fluentd -c Chapter3/Fluentd/rotating-file-read-regex.conf
```
As Fluentd outputs the processed stream of events to console, you will see entries like this:
```bash
2020-04-30 23:29:47.000000000 +0100 simpleFile: {"level":"","class":"INFO","line":"50","iteration":"1","msg":"The truth is out there. Anybody got the URL"}
```
Each line printed by Fluentd to the console will take the date and time of the log event, the time zone offset, the event tag, and the payload as a correctly structured JSON, with the time omitted. Notice how the nanoseconds are all 0. This is because we have given Fluentd a log time that does not have nanosecond precision; therefore, that part of the timestamp is left at 0.

More importantly, you will note that all the JSON values are quoted, so they will be treated as strings. This may not be an issue. But having come this far, it would be a shame not to define the data types correctly. It may well enable downstream activities, such as extrapolating additional meaning, to be more effective. The defining of nonstring data types is straightforward. We need to add the attribute types within the parser construct, which takes a comma-separated list with each defined value described in the format ``name:type``. In our use case, we would want to add ``types line:integer,iteration:intege``r. The complete list of types supported are

* ``string``—Can be defined explicitly, but is the default type
* ``bool``—Boolean
* ``integer``—Representation for any whole number (i.e., no decimal places)
* ``float``—Represents any decimal number
* ``time``—Converts the value into the way that Fluentd represents time internally.
We can extend this to describe how the time should translate. For example:
    * date:time:%d/%b/%Y:%H:%M—Defining the formatting of the representation
    * date:time:unixtime—Timer from 1 Jan 1970 in integer format
    * date:time:float—The same epoch point, but the number is as float
* ``array``—A sequence of values of the same type (e.g., all strings, all integers)

The handling of the array requires the values to have a delimiter to separate each value. The delimiter by default is a comma, but it can be changed by adding a colon and the delimiter character. For example, a comma-delimited array could be defined as ``myList:array``. But if I wanted to replace the delimiter with a hash, then the expression would be ``myList:array:#``.

The last manipulation of the JSON involves whether we would like the date timestamp to be included in the JSON; after all, it was in the body of the log event. This can easily be done by adding ``keep_time_key`` true to the parser attributes. We can add the changes described (although the provided configuration has these values ready but commented out, so you could just uncomment them and rerun the simulator and Fluentd as before). As a result of these changes, the log entries will appear like this:
```bash
2020-05-01 00:14:25.000000000 +0100 simpleFile: {"time":"2020-05-01--00:14:25","level":"","class":"INFO","line":75,"iteration":1,"msg":"I started a band called 999 megabytes we still havent gotten a gig"}
```
If you look at the JSON body now, our numeric elements are no longer in quotes, and the timestamp appears in the JSON payload.

> **⚠ NOTE:** Evaluating/checking Regex expressions
> Regex expressions can be challenging; the last thing we want to have to do is run logs throughout the Fluentd environment to determine whether the expression is complete or not. To this end, Fluentd UI configuration for tail supports Regex validation.
> In addition, there is a free web tool called ``Fluentular`` (https://fluentular.herokuapp.com/) that will allow you to develop and test expressions.
> Some IDEs, such as Microsoft’s Visual Studio Code, have Regex tools to help visualize the Regex being built—for example, Regexp Explain (http://mng.bz/q2YA). The completed Regex can be seen in the following figure.
> If you look carefully, you will note that the ``?<element name>`` is missing; however, it is easy to see where these pieces need to be added, as the core parts have been grouped. If the grouping is used, it becomes easy to port the expression into Fluentd and add the elements.

### 3.4.4 Putting parser configuration into action
This exercise is designed to allow you to work with the parser. The simulator configuration ``Chapter3/SimulatorConfig/jul-log-file2-exercise.properties`` has some differences to the previous worked example. Copy the Fluentd configuration file used to illustrate the parser (``Chapter3/Fluentd/rotating-file-readregex.conf``). Then, modify the parser expression so that all the input values are
properly represented as JSON elements of the log event, rather than just the payload as defaulted by Fluentd. A variation of the log simulator configuration can be run using the command
```bash
groovy LogSimulator.groovy Chapter3/SimulatorConfig/jul-log-file2-exercise.properties ./TestData/medium-source.txt
```
Run the revised Fluentd configuration and determine whether your changes have been effective.
##### ANSWER
The parser configuration should appear like the code shown in the following listing.

Listing 3.7 Chapter3/ExerciseResults/rotating-file-read-regex-Answer.conf
```conf
  <parse>
    @type regexp
    expression /(?<time>\S+)\s(?<level>[A-Z]*)\s*(?<class>\S+)[^\d]*(?<iteration>[\d]*)\-(?<line>[\d]*)\][\s]+\{"event":"(?<msg>.*(?="\,))/
    time_format %Y-%m-%d--%T
    time_key time
    types line:integer,iteration:integer
    keep_time_key true
  </parse>
```
A complete configuration file is provided at ``Chapter3/ExerciseResults/rotating-file-read-regex-Answer.conf``.
