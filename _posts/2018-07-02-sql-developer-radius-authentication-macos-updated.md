---
layout: post
title: RADIUS authentication in SQL Developer on MacOS (updated)
lead: Configure SQL Developer to use RADIUS (two-factor/MFA or other reasons) for authentication into an Oracle instance on modern MacOS.
---

This post is an update to [this original overview](/2017/02/sql-developer-radius-authentication-macos/),
which details configuring SQL Developer to perform RADIUS authentication on MacOS.
This updated post contains newer versions numbers and simpler instructions.


### 1. Install instantclient

Download [instantclient 12.2.0.1.0](http://www.oracle.com/technetwork/topics/intel-macsoft-096467.html) from Oracle.
You’ll only need the basic package. Unzip this package and place it in an accessible location.
I install my instant client in `~/lib/instantclient_12_2`. You can install yours
wherever you like.

Homebrew users: the Homebrew packages available strip out the `.jar` files which
confuses SQL Developer; download this library directly from Oracle.

### 2. sqlnet.ora

The OCI driver properties can be configured via a special file called `sqlnet.ora`. The property
we're interested in is [SQLNET.AUTHENTICATION_SERVICES](https://docs.oracle.com/cd/E11882_01/network.112/e10835/sqlnet.htm#NETRF2035).

Create a `sqlnet.ora` file and place it in a directory you'll reference later as `TNS_ADMIN`. I put mine alongside the
instantclient directory in step 1: `~/lib/sqlnet.ora`.

Make sure the `sqlnet.ora` file has an entry for `radius`:

```
SQLNET.AUTHENTICATION_SERVICES=(none,radius)
```

### 3. Mangle the java.library.path property

Modern MacOSes—version 10.11 (El Capitan) and newer—have a feature called System Integrity Protection (SIP)
which ignores the `DYLD_LIBRARY_PATH` environment variable. This is how the instantclient path
is typically revealed to SQL Developer. SQL Developer still looks at this environment variable even though we
can no longer use it. It's simply not an option for setting this path.

Instead we can clobber the `java.library.path` via a not-so-invasive configuration file, `~/.sqldeveloper/<VERSION>/product.conf`.
Open this file and set `java.library.path` to your instantclient path (the one you chose in step 1) at the very bottom.
If you chose the same location I did, yours should look like this (replace `<YOUR_USERNAME>` with your username):

```
AddVMOption -Djava.library.path=/Users/<YOUR_USERNAME>/lib/instantclient_12_2
```

### 4. Set the TNS_ADMIN path for GUI applications

You must set the TNS_ADMIN environment variable to make this work. Note that this needs to be set in
the GUI environment via `launchctl`—simply setting the environment variable
in a terminal will not work. Unfortunately, values set via `launchctl` don't persist between sessions
(if you logout or restart, the value is lost).

In order to set this value upon start, we'll create the file `~/.sqldeveloper/env.sh`.
SQL Developer uses this file to set environment variables upon startup.

Create/open this file and add a reference to your `TNS_ADMIN` location:

```
launchctl setenv TNS_ADMIN /Users/<YOUR_USERNAME>/lib
```

Note that `TNS_ADMIN` should point to a directory—not a file.

### 4. Configure SQL Developer to use OCI

Now, fire up SQL Developer. Navigate to *Oracle SQL Developer* > *Preferences* and choose *Database* > *Advanced*:

![SQL Developer screenshot](/images/instantclient1.png)

Check the *Use Oracle Client* and *Use OCI/Thick driver* checkboxes.

Then, click *Configure* and enter the location of your instantclient:

![SQL Developer screenshot](/images/instantclient2.png)

Make sure you change the *Client type* to `Instant Client`.

You may click the *Test* button here if you want, though I've found that its output is not always useful (failures here do not necessarily
indicate a broken configuration).

### 5. Verify

In SQL Developer, go to *Oracle SQL Developer* > *About Oracle SQL Developer*. Click on the *Properties* tab
and search for the `sqldeveloper.oci.available` property. If the value is `true`, everything is set up correctly.
You should now be able to authenticate using RADIUS!

## References

- [https://community.oracle.com/thread/3995451](https://community.oracle.com/thread/3995451)
- [http://ilmarkerm.blogspot.com/2015/03/setting-up-sql-developer-with-instant.html](http://ilmarkerm.blogspot.com/2015/03/setting-up-sql-developer-with-instant.html)
