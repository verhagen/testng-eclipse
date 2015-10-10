# User documentation

The documentation for this plug-in can be found at http://testng.org/doc/eclipse.html

# Build

## Version number

Set the version number with `scripts/set-version`, e.g.:

```
scripts/set-version 6.8.22-SNAPSHOT
```

Using `SNAPSHOT` version numbers will generate UTC timestamped plugin-numbers, e.g. 6.8.22.201505030200 so you should only increase the version number if you are including a higher version of `testng.jar`.

## Building

Once the version is correct, build the Eclipse plug-in as follows:

```
mvn -e clean package
```

The update site package will be generated at `testng-eclipse-update-site/target/site.zip`

# Developer

In Eclipse, select _Import / Existing Project_ and point the dialog to
this directory. Then you can just create a new Eclipse application
launch to run the plug-in.

The runner view is called TestRunnerViewPart and it receives the test
results from the remote TestNG process. Every new result is passed to
```postTestResult()``` which in turn, passes this result to each tab by
calling their ```updateTestResult()``` method.

The tab's logic is in ```AbstractTab```, which calculates a unique id for
each test result and then either creates or updates the corresponding
node in the tree. Each node is associated with an instance of an
```ITreeItem``` (store in its data map) which contains all the necessary
information to display the label, its image, etc...

The tests are run by a subclass of ```TestNG``` called ```RemoteTestNG```. The
Eclipse client forks the ```RemoteTestNG``` process and adds itself as a
listener. The difference is that this listener will pass the test
results over the network using a serialization based protocol that can
be found in the ```strprotocol``` package.

The plug-in understands two system properties, which you can define as
VM arguments in the launch dialog:

```-Dtestng.eclipse.verbose
``` 

This will cause both the Eclipse client and RemoteTestNG to issue a
more verbose output.

```-Dtestng.eclipse.debug
```

Use this flag if you need to debug and break into RemoteTestNG. In
this case, you need to start the RemoteTestNG process youself as a
regular Java application and with the "-debug" flag. Then start the
Eclipse client with this system property, and then the two processes
will communicate on a hardcoded port, 12345 (as opposed to the random
port which they usually use) and through a hardcoded XML file
(```"${java.io.tmpdir}/testng-customsuite.xml"```).

Now that you launched both processes yourself, you can set up break
point and inspect variables on either.


# Protocol

When a new run is launched, TestNGLaunchConfigurationDelegate creates
a VMRunnerConfigurationClient that launches RemoteTestNG with a host,
a port and an XML file. Then Eclipse listens on this host and port.

The base class that provides the basic listening functions is
AbstractRemoteTestRunnerClient, which is defined in TestNG. The
Eclipse plug-in subclasses this class with an
EclipseTestRunnerClient. TestRunnerViewPart creates an instance of
this class and then calls startListening() on it.

Whenever a new message is received, AbstractRemoteTestRunnerClient
looks up the type of the message and then calls the subclass's
corresponding method:

```SUITE -> notifyStart(GenericMessage)
TEST -> notifySuiteEvents(SuiteMessage)
TEST_RESULT -> notifyTestEvents(TestMessage)
other -> notifyResultEvents(TestResultMessage)```

RemoteTestNG starts by opening a connection to the port passed on the
command line and when it succeeds, runs the suites and uses listeners
to send messages to the Eclipse client.

All these messages implement IStringMessage and they are of several kinds:

GenericMessage: general information message (such as an initial notification of the number of suites/tests)

- TestMessage
- SuiteMessage
- TestResultMessage
