The Great Debugging of the `signal-cli` Daemon D-Bus Integration
================================================================

[signal-cli](https://github.com/AsamK/signal-cli) is a command-line client for the [Signal messaging service](https://www.signal.org/).

It has been recently ported to FreeBSD ([net-im/signal-cli](http://freshports.org/net-im/signal-cli)\). Most of its features work as expected except its integration with [D-Bus](https://www.freedesktop.org/wiki/Software/dbus/), which is necessary for UI wrappers for signal-cli like [scli](https://github.com/isamert/scli).

Expected behavior
-----------------

1.	The service starts without any issues:

	```
	service signal_cli start
	```

2.	The user can send messages via the signal-cli daemon:

	```
	signal-cli --dbus-system send -m "Message" +00123123123
	```

3.	The user is able to start scli without any issues.

Enviroment setup
----------------

-	Install the port.
-	Link (or register) signal-cli to your Signal account. Do it as the `signal-user` from `/var/lib/signal-cli`.

Current challanges
------------------

### org.freedesktop.dbus.exceptions.DBusException: Failed to connect to bus Failed to auth

```console
$ export DISPLAY=0 
$ export JAVA_OPTS="-Djava.library.path=/usr/local/lib"
$ signal-cli -u +00123456789 daemon
org.freedesktop.dbus.exceptions.DBusException: Failed to connect to bus Failed to auth
        at org.freedesktop.dbus.DBusConnection.<init>(DBusConnection.java:304)
        at org.freedesktop.dbus.DBusConnection.getConnection(DBusConnection.java:282)
        at org.asamk.signal.commands.DaemonCommand.handleCommand(DaemonCommand.java:50)
        at org.asamk.signal.Main.handleCommands(Main.java:126)
        at org.asamk.signal.Main.main(Main.java:61)
```

In `net-im/signal-cli/work/signal-cli-0.6.5/src/main/java/org/asamk/signal/commands/DaemonCommand.java` (around line 50):

```java
        DBusConnection conn = null;
        try {
            try {
                int busType;
                if (ns.getBoolean("system")) {
                    busType = DBusConnection.SYSTEM;
                } else {
                    busType = DBusConnection.SESSION;
                }
                conn = DBusConnection.getConnection(busType);
                conn.exportObject(SIGNAL_OBJECTPATH, m);
                conn.requestBusName(SIGNAL_BUSNAME);
            } catch (UnsatisfiedLinkError e) {
                System.err.println("Missing native library dependency for dbus service: " + e.getMessage());
                return 1;
            } catch (DBusException e) {
                e.printStackTrace();
                return 2;
            }
```

In `devel/dbus-java/work/dbus-java-2.7/org/freedesktop/dbus/DBusConnection.java` (around line 288):

```java
   @SuppressWarnings("unchecked")
   private DBusConnection(String address) throws DBusException
   {
      super(address);
      busnames = new Vector<String>();

      synchronized (_reflock) {
         _refcount = 1; 
      }
   
      try {
         transport = new Transport(addr, AbstractConnection.TIMEOUT);
			connected = true;
      } catch (IOException IOe) {
         if (EXCEPTION_DEBUG && Debug.debug) Debug.print(Debug.ERR, IOe);            
         disconnect();
         throw new DBusException(_("Failed to connect to bus ")+IOe.getMessage());
      } catch (ParseException Pe) {
         if (EXCEPTION_DEBUG && Debug.debug) Debug.print(Debug.ERR, Pe);            
         disconnect();
         throw new DBusException(_("Failed to connect to bus ")+Pe.getMessage());
      }

      // start listening for calls
      listen();

      // register disconnect handlers
      DBusSigHandler h = new _sighandler();
      addSigHandlerWithoutMatch(org.freedesktop.DBus.Local.Disconnected.class, h);
      addSigHandlerWithoutMatch(org.freedesktop.DBus.NameAcquired.class, h);

      // register ourselves
      _dbus = getRemoteObject("org.freedesktop.DBus", "/org/freedesktop/DBus", DBus.class);
      try {
         busnames.add(_dbus.Hello());
      } catch (DBusExecutionException DBEe) {
         if (EXCEPTION_DEBUG && Debug.debug) Debug.print(Debug.ERR, DBEe);
         throw new DBusException(DBEe.getMessage());
      }
   }
```

Let's run signal-cli with some more debug support.

First, some preparation:

1.	Build a debug version of `devel/dbus-java` (e.g., `cd ports/devel/dbus-java && make WITH_DEBUG=yes clean reinstall`).
2.	Modify the class path in `signal-cli` script. Replace the following JAR files:

	-	`/usr/local/share/signal-cli/lib/debug-1.1.1.jar`
	-	`/usr/local/share/signal-cli/lib/dbus-java-2.7.0.jar

	with:

	-	`/usr/local/share/java/classes/dbus-2.7.jar`
	-	`/usr/local/share/java/classes/debug-enable.jar`

3.	Add the following code to the `signal-cli` script to gain additional debugging infomation:

	```
	DBUS_JAVA_EXCEPTION_DEBUG=yes
	```

	See `/usr/local/share/doc/dbus-java/INSTALL` for more details.

4.	Create a `debug.conf` file in the directory from which you start `signal-cli`:

	```console
	$ echo ALL = ALL > debug.conf
	```

	See `/usr/local/share/doc/dbus-java/INSTALL` for more details.

Now, start `signal-cli`:

```console
# chroot -u signal-cli / env PS1='\w$ ' /bin/sh
/$ cd /var/lib/signal-cli
/var/lib/signal-cli$ signal-cli -u +00123456789 --config $PWD daemon --system
[org.freedesktop.dbus.AbstractConnection.<clinit>()] Debugging of internal exceptions enabled
[org.freedesktop.dbus.AbstractConnection.<clinit>()] Loading debug config file: debug.conf
[org.freedesktop.dbus.DBusConnection.getConnection()] Getting bus connection for unix:path=/var/run/dbus/system_bus_socket: null
[org.freedesktop.dbus.DBusConnection.getConnection()] Creating new bus connection to: unix:path=/var/run/dbus/system_bus_socket
[org.freedesktop.dbus.MethodTuple.<init>()] new MethodTuple(Ping, )
[org.freedesktop.dbus.Marshalling.recursiveGetDBusType()] Converted Java type: class java.lang.String to D-Bus Type: s
[org.freedesktop.dbus.MethodTuple.<init>()] new MethodTuple(Introspect, )
[org.freedesktop.dbus.EfficientQueue.shrink()] Shrinking
[org.freedesktop.dbus.BusAddress.<init>()] Parsing bus address: unix:path=/var/run/dbus/system_bus_socket
[org.freedesktop.dbus.BusAddress.<init>()] Transport type: unix
[org.freedesktop.dbus.BusAddress.<init>()] Transport options: {path=/var/run/dbus/system_bus_socket}
[org.freedesktop.dbus.Transport.connect()] Connecting to unix: {path=/var/run/dbus/system_bus_socket}
[org.freedesktop.dbus.Transport$SASL.auth()] AUTH state: 0
[org.freedesktop.dbus.Transport$SASL.send()] sending: AUTH

[org.freedesktop.dbus.Transport$SASL.auth()] AUTH state: 1
[org.freedesktop.dbus.Transport$SASL.receive()] received: REJECTED EXTERNAL
[org.freedesktop.dbus.Transport$SASL$Command.<init>()] Creating command from: [REJECTED, EXTERNAL]
[org.freedesktop.dbus.Transport$SASL$Command.<init>()] Created command: Command(3, 1, null, null)
[org.freedesktop.dbus.Transport$SASL.send()] sending: AUTH EXTERNAL 323438

[org.freedesktop.dbus.Transport$SASL.auth()] AUTH state: 1
[org.freedesktop.dbus.Transport$SASL.receive()] received: REJECTED EXTERNAL
[org.freedesktop.dbus.Transport$SASL$Command.<init>()] Creating command from: [REJECTED, EXTERNAL]
[org.freedesktop.dbus.Transport$SASL$Command.<init>()] Created command: Command(3, 1, null, null)
[org.freedesktop.dbus.DBusConnection.<init>()] java.io.IOException: Failed to auth
    at org.freedesktop.dbus.Transport.connect(Unknown Source)
    at org.freedesktop.dbus.Transport.<init>(Unknown Source)
    at org.freedesktop.dbus.DBusConnection.<init>(Unknown Source)
    at org.freedesktop.dbus.DBusConnection.getConnection(Unknown Source)
    at org.asamk.signal.commands.DaemonCommand.handleCommand(DaemonCommand.java:50)
    at org.asamk.signal.Main.handleCommands(Main.java:126)
    at org.asamk.signal.Main.main(Main.java:61)
[org.freedesktop.dbus.DBusConnection.disconnect()] Disconnecting DBusConnection
[org.freedesktop.dbus.Message.<init>()] Creating message with serial 1
[org.freedesktop.dbus.Message.append()] Appending sig: yyyy data: [66, 3, 0, 1]
[org.freedesktop.dbus.Message.append()] Appending item: 0 y 0
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 66
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 4 0 4 1
[org.freedesktop.dbus.Message.append()] Appending item: 1 y 1
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 3
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 3 1 4 1
[org.freedesktop.dbus.Message.append()] Appending item: 2 y 2
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 0
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 2 2 4 1
[org.freedesktop.dbus.Message.append()] Appending item: 3 y 3
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 1
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 1 3 4 1
[org.freedesktop.dbus.Message.append()] Appending sig: ua(yv) data: [1, [[4, [s, org.freedesktop.DBus.Local.Disconnected]], [5, [u, 0]], [6, [s, org.freedesktop.DBus.Local]], [8, [g, s]]]]
[org.freedesktop.dbus.Message.append()] Appending item: 0 u 0
[org.freedesktop.dbus.Message.appendone()] 8
[org.freedesktop.dbus.Message.appendone()] Appending type: u value: 1
[org.freedesktop.dbus.Message.pad()] padding for u
[org.freedesktop.dbus.Message.pad()] 0 4 8 4
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 1 to 00 00 00 01 
[org.freedesktop.dbus.Message.append()] Appending item: 1 a 1
[org.freedesktop.dbus.Message.appendone()] 12
[org.freedesktop.dbus.Message.appendone()] Appending type: a value: [Ljava.lang.Object;@5d0a1059
[org.freedesktop.dbus.Message.pad()] padding for a
[org.freedesktop.dbus.Message.pad()] 0 4 12 4
[org.freedesktop.dbus.Message.appendone()] Appending array: [[4, [s, org.freedesktop.DBus.Local.Disconnected]], [5, [u, 0]], [6, [s, org.freedesktop.DBus.Local]], [8, [g, s]]]
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 4 16 8
[org.freedesktop.dbus.Message.appendone()] 16
[org.freedesktop.dbus.Message.appendone()] Appending type: ( value: [Ljava.lang.Object;@485966cc
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 4 16 8
[org.freedesktop.dbus.Message.appendone()] 16
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 4
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 0 4 16 1
[org.freedesktop.dbus.Message.appendone()] 17
[org.freedesktop.dbus.Message.appendone()] Appending type: v value: [Ljava.lang.Object;@1de76cc7
[org.freedesktop.dbus.Message.pad()] padding for v
[org.freedesktop.dbus.Message.pad()] 0 4 17 1
[org.freedesktop.dbus.Message.appendone()] 17
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: s
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 4 17 1
[org.freedesktop.dbus.Message.appendone()] 20
[org.freedesktop.dbus.Message.appendone()] Appending type: s value: org.freedesktop.DBus.Local.Disconnected
[org.freedesktop.dbus.Message.pad()] padding for s
[org.freedesktop.dbus.Message.pad()] 0 3 20 4
[org.freedesktop.dbus.Message.appendone()] Appending String of length 39
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 39 to 00 00 00 27 
[org.freedesktop.dbus.Message.appendone()] 64
[org.freedesktop.dbus.Message.appendone()] Appending type: ( value: [Ljava.lang.Object;@54bff557
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 3 64 8
[org.freedesktop.dbus.Message.appendone()] 64
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 5
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 0 3 64 1
[org.freedesktop.dbus.Message.appendone()] 65
[org.freedesktop.dbus.Message.appendone()] Appending type: v value: [Ljava.lang.Object;@593aaf41
[org.freedesktop.dbus.Message.pad()] padding for v
[org.freedesktop.dbus.Message.pad()] 0 3 65 1
[org.freedesktop.dbus.Message.appendone()] 65
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: u
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 3 65 1
[org.freedesktop.dbus.Message.appendone()] 68
[org.freedesktop.dbus.Message.appendone()] Appending type: u value: 0
[org.freedesktop.dbus.Message.pad()] padding for u
[org.freedesktop.dbus.Message.pad()] 0 3 68 4
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 0 to 00 00 00 00 
[org.freedesktop.dbus.Message.appendone()] 72
[org.freedesktop.dbus.Message.appendone()] Appending type: ( value: [Ljava.lang.Object;@5a56cdac
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 3 72 8
[org.freedesktop.dbus.Message.appendone()] 72
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 6
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 0 3 72 1
[org.freedesktop.dbus.Message.appendone()] 73
[org.freedesktop.dbus.Message.appendone()] Appending type: v value: [Ljava.lang.Object;@7c711375
[org.freedesktop.dbus.Message.pad()] padding for v
[org.freedesktop.dbus.Message.pad()] 0 3 73 1
[org.freedesktop.dbus.Message.appendone()] 73
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: s
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 3 73 1
[org.freedesktop.dbus.Message.appendone()] 76
[org.freedesktop.dbus.Message.appendone()] Appending type: s value: org.freedesktop.DBus.Local
[org.freedesktop.dbus.Message.pad()] padding for s
[org.freedesktop.dbus.Message.pad()] 0 3 76 4
[org.freedesktop.dbus.Message.appendone()] Appending String of length 26
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 26 to 00 00 00 1a 
[org.freedesktop.dbus.Message.appendone()] 107
[org.freedesktop.dbus.Message.appendone()] Appending type: ( value: [Ljava.lang.Object;@57cf54e1
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 3 107 8
[org.freedesktop.dbus.Message.pad()] 0 3 112 5
[org.freedesktop.dbus.Message.ensureBuffers()] Resizing 18
[org.freedesktop.dbus.Message.appendone()] 112
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 8
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 0 3 112 1
[org.freedesktop.dbus.Message.appendone()] 113
[org.freedesktop.dbus.Message.appendone()] Appending type: v value: [Ljava.lang.Object;@5b03b9fe
[org.freedesktop.dbus.Message.pad()] padding for v
[org.freedesktop.dbus.Message.pad()] 0 3 113 1
[org.freedesktop.dbus.Message.appendone()] 113
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: g
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 3 113 1
[org.freedesktop.dbus.Message.appendone()] 116
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: s
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 3 116 1
[org.freedesktop.dbus.Message.appendone()] start: 16 end: 119 length: 103
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 103 to 00 00 00 67 
[org.freedesktop.dbus.Message.pad()] padding for 
[org.freedesktop.dbus.Message.pad()] 0 3 119 8
[org.freedesktop.dbus.Message.pad()] 0 3 120 1
[org.freedesktop.dbus.Message.append()] Appending sig: s data: [Disconnected]
[org.freedesktop.dbus.Message.append()] Appending item: 0 s 0
[org.freedesktop.dbus.Message.appendone()] 120
[org.freedesktop.dbus.Message.appendone()] Appending type: s value: Disconnected
[org.freedesktop.dbus.Message.pad()] padding for s
[org.freedesktop.dbus.Message.pad()] 0 3 120 4
[org.freedesktop.dbus.Message.appendone()] Appending String of length 12
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 12 to 00 00 00 0c 
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 17 to 00 00 00 11 
[org.freedesktop.dbus.AbstractConnection.disconnect()] Sending disconnected signal
[org.freedesktop.dbus.Message.<init>()] Creating message with serial 2
[org.freedesktop.dbus.Message.append()] Appending sig: yyyy data: [66, 4, 0, 1]
[org.freedesktop.dbus.Message.append()] Appending item: 0 y 0
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 66
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 4 0 4 1
[org.freedesktop.dbus.Message.append()] Appending item: 1 y 1
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 4
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 3 1 4 1
[org.freedesktop.dbus.Message.append()] Appending item: 2 y 2
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 0
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 2 2 4 1
[org.freedesktop.dbus.Message.append()] Appending item: 3 y 3
[org.freedesktop.dbus.Message.appendone()] 4
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 1
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 1 3 4 1
[org.freedesktop.dbus.Message.append()] Appending sig: ua(yv) data: [3, [[1, [o, /]], [2, [s, org.freedesktop.DBus.Local]], [3, [s, Disconnected]]]]
[org.freedesktop.dbus.Message.append()] Appending item: 0 u 0
[org.freedesktop.dbus.Message.appendone()] 8
[org.freedesktop.dbus.Message.appendone()] Appending type: u value: 3
[org.freedesktop.dbus.Message.pad()] padding for u
[org.freedesktop.dbus.Message.pad()] 0 4 8 4
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 3 to 00 00 00 03 
[org.freedesktop.dbus.Message.append()] Appending item: 1 a 1
[org.freedesktop.dbus.Message.appendone()] 12
[org.freedesktop.dbus.Message.appendone()] Appending type: a value: [Ljava.lang.Object;@3c7f66c4
[org.freedesktop.dbus.Message.pad()] padding for a
[org.freedesktop.dbus.Message.pad()] 0 4 12 4
[org.freedesktop.dbus.Message.appendone()] Appending array: [[1, [o, /]], [2, [s, org.freedesktop.DBus.Local]], [3, [s, Disconnected]]]
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 4 16 8
[org.freedesktop.dbus.Message.appendone()] 16
[org.freedesktop.dbus.Message.appendone()] Appending type: ( value: [Ljava.lang.Object;@194bcebf
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 4 16 8
[org.freedesktop.dbus.Message.appendone()] 16
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 1
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 0 4 16 1
[org.freedesktop.dbus.Message.appendone()] 17
[org.freedesktop.dbus.Message.appendone()] Appending type: v value: [Ljava.lang.Object;@17497425
[org.freedesktop.dbus.Message.pad()] padding for v
[org.freedesktop.dbus.Message.pad()] 0 4 17 1
[org.freedesktop.dbus.Message.appendone()] 17
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: o
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 4 17 1
[org.freedesktop.dbus.Message.appendone()] 20
[org.freedesktop.dbus.Message.appendone()] Appending type: o value: /
[org.freedesktop.dbus.Message.pad()] padding for o
[org.freedesktop.dbus.Message.pad()] 0 3 20 4
[org.freedesktop.dbus.Message.appendone()] Appending String of length 1
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 1 to 00 00 00 01 
[org.freedesktop.dbus.Message.appendone()] 26
[org.freedesktop.dbus.Message.appendone()] Appending type: ( value: [Ljava.lang.Object;@f0da945
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 3 26 8
[org.freedesktop.dbus.Message.pad()] 0 3 32 6
[org.freedesktop.dbus.Message.appendone()] 32
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 2
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 0 3 32 1
[org.freedesktop.dbus.Message.appendone()] 33
[org.freedesktop.dbus.Message.appendone()] Appending type: v value: [Ljava.lang.Object;@4803b726
[org.freedesktop.dbus.Message.pad()] padding for v
[org.freedesktop.dbus.Message.pad()] 0 3 33 1
[org.freedesktop.dbus.Message.appendone()] 33
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: s
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 3 33 1
[org.freedesktop.dbus.Message.appendone()] 36
[org.freedesktop.dbus.Message.appendone()] Appending type: s value: org.freedesktop.DBus.Local
[org.freedesktop.dbus.Message.pad()] padding for s
[org.freedesktop.dbus.Message.pad()] 0 3 36 4
[org.freedesktop.dbus.Message.appendone()] Appending String of length 26
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 26 to 00 00 00 1a 
[org.freedesktop.dbus.Message.appendone()] 67
[org.freedesktop.dbus.Message.appendone()] Appending type: ( value: [Ljava.lang.Object;@ffaa6af
[org.freedesktop.dbus.Message.pad()] padding for (
[org.freedesktop.dbus.Message.pad()] 0 3 67 8
[org.freedesktop.dbus.Message.pad()] 0 3 72 5
[org.freedesktop.dbus.Message.ensureBuffers()] Resizing 16
[org.freedesktop.dbus.Message.appendone()] 72
[org.freedesktop.dbus.Message.appendone()] Appending type: y value: 3
[org.freedesktop.dbus.Message.pad()] padding for y
[org.freedesktop.dbus.Message.pad()] 0 3 72 1
[org.freedesktop.dbus.Message.appendone()] 73
[org.freedesktop.dbus.Message.appendone()] Appending type: v value: [Ljava.lang.Object;@53ce1329
[org.freedesktop.dbus.Message.pad()] padding for v
[org.freedesktop.dbus.Message.pad()] 0 3 73 1
[org.freedesktop.dbus.Message.appendone()] 73
[org.freedesktop.dbus.Message.appendone()] Appending type: g value: s
[org.freedesktop.dbus.Message.pad()] padding for g
[org.freedesktop.dbus.Message.pad()] 0 3 73 1
[org.freedesktop.dbus.Message.appendone()] 76
[org.freedesktop.dbus.Message.appendone()] Appending type: s value: Disconnected
[org.freedesktop.dbus.Message.pad()] padding for s
[org.freedesktop.dbus.Message.pad()] 0 3 76 4
[org.freedesktop.dbus.Message.appendone()] Appending String of length 12
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 12 to 00 00 00 0c 
[org.freedesktop.dbus.Message.appendone()] start: 16 end: 93 length: 77
[org.freedesktop.dbus.Message.marshallint()] Marshalled int 77 to 00 00 00 4d 
[org.freedesktop.dbus.Message.pad()] padding for 
[org.freedesktop.dbus.Message.pad()] 0 3 93 8
[org.freedesktop.dbus.Message.pad()] 0 3 96 3
[org.freedesktop.dbus.AbstractConnection.handleMessage()] Handling incoming signal: Disconnected(0,3) { Path=>/, Interface=>org.freedesktop.DBus.Local, Member=>Disconnected } { }
[org.freedesktop.dbus.AbstractConnection.disconnect()] Disconnecting Abstract Connection
org.freedesktop.dbus.exceptions.DBusException: Failed to connect to bus Failed to auth
    at org.freedesktop.dbus.DBusConnection.<init>(Unknown Source)
    at org.freedesktop.dbus.DBusConnection.getConnection(Unknown Source)
    at org.asamk.signal.commands.DaemonCommand.handleCommand(DaemonCommand.java:50)
    at org.asamk.signal.Main.handleCommands(Main.java:126)
    at org.asamk.signal.Main.main(Main.java:61)
```

Troubleshooting Java Exceptions
-------------------------------

### org.freedesktop.dbus.exceptions.DBusException: Cannot Resolve Session Bus Address

```console
$ signal-cli -u +00123456789 daemon
org.freedesktop.dbus.exceptions.DBusException: Cannot Resolve Session Bus Address
        at org.freedesktop.dbus.DBusConnection.getConnection(DBusConnection.java:267)
        at org.asamk.signal.commands.DaemonCommand.handleCommand(DaemonCommand.java:50)
        at org.asamk.signal.Main.handleCommands(Main.java:126)
        at org.asamk.signal.Main.main(Main.java:61)
```

#### Solution

The DISPLAY environment variable has to omit the leading ":", e.g., "0" instead of ":0". Otherwise, signal-cli tries to open an invalid D-BUS session bus file, e.g., `~/.dbus/session-bus/0123456789abcdef0123456789abcdef-:0` instead of `~/.dbus/session-bus/0123456789abcdef0123456789abcdef-0`.

### org.freedesktop.dbus.exceptions.DBusException: Failed to connect to bus unknown address type 'unix

```console
$ DISPLAY=0 signal-cli -u +00123456789 daemon
org.freedesktop.dbus.exceptions.DBusException: Failed to connect to bus unknown address type 'unix
        at org.freedesktop.dbus.DBusConnection.<init>(DBusConnection.java:304)
        at org.freedesktop.dbus.DBusConnection.getConnection(DBusConnection.java:282)
        at org.asamk.signal.commands.DaemonCommand.handleCommand(DaemonCommand.java:50)
        at org.asamk.signal.Main.handleCommands(Main.java:126)
        at org.asamk.signal.Main.main(Main.java:61)
```

#### Solution

Unquote the value of the `DBUS_SESSION_BUS_ADDRESS` variable in `~/.dbus/session-bus/0123456789abcdef0123456789abcdef-0`. For example:

```
DBUS_SESSION_BUS_ADDRESS='unix:path=/tmp/dbus-ABCDEFGHIJ,guid=fedcba9876543210fedcba9876543210'
```

should be

```
DBUS_SESSION_BUS_ADDRESS=unix:path=/tmp/dbus-ABCDEFGHIJ,guid=fedcba9876543210fedcba9876543210
```

### Missing native library dependency for dbus service: no unix-java in java.library.path

```console
$ export DISPLAY=0 
$ signal-cli -u +00123456789 daemon
Missing native library dependency for dbus service: no unix-java in java.library.path
```

`signal-cli` cannot find the share library provided by libmatthew. A proper path can be set via `JAVA_OPTS`:

```sh
export JAVA_OPTS="-Djava.library.path=/usr/local/lib"
```

General notes
-------------

Use the following command to list all available message buses:

```
# Source: https://unix.stackexchange.com/a/46309
dbus-send --print-reply --dest=org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus.ListNames
```

Send a Signal message via the signal-cli D-Bus daemon:

```
dbus-send --session --type=method_call --print-reply --dest="org.asamk.Signal" /org/asamk/Signal org.asamk.Signal.sendMessage string:MessageText array:string: string:RECIPIENT \`\`\`
```

`qdbus` (from `devel/qt5-qdbus`) can be used to view avaiable interfaces to D-Bus.

<!-- vim: softtabstop=8 shiftwidth=8 tabstop=8 noexpandtab
-->
