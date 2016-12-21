---
layout: post
title: Postgres foreign data wrapper for Oracle (oracle_fdw) on MacOS
lead: The Ecto adapter landscape currently lacks an Oracle option, but you can use
      Postgres' foreign data wrapper feature to connect to a remote Oracle database.
      This posts will help you get oracle_fdw installed on
      newer versions of MacOS running SIP (System Integrity Protection).
---

The Ecto adapter landscape currently lacks an Oracle option. If you're itching
to try out Elixir and/or Phoenix but really need to connect to an Oracle
database to do anything interesting, you may want to give Laurenz Albe's
[oracle_fdw](https://github.com/laurenz/oracle_fdw) project a try.
oracle_fdw implements a foreign data wrapper for Postgres that allows you to
connect to, read from, and manipulate remote Oracle databases.

Oracle_fdw relies on `DYLD_LIBRARY_PATH` to properly link Oracle libraries at
runtime, but MacOS version 10.11 (El Capitan) and newer ignore `DYLD_LIBRARY_PATH` as
part of its System Integrity Protection (SIP) feature. These steps will help you both
build and install oracle_fdw on MacOS.

### 1. Install Postgres

The easiest way to install Postgres is homebrew: `brew install postgres`. Once installed
it can be started with `brew services start postgres`.

### 2. Install instantclient

Download an [instantclient](http://www.oracle.com/technetwork/topics/intel-macsoft-096467.html)
from Oracle. You'll need both the basic package and the SDK package.
Out of the box, oracle_fdw expects that you'll be using version 11.2. If you want
to use 12.1, you'll need to create some symlinks to trick it.

I install my instant client in `~/lib/instantclient_11_2`. You can install yours
wherever you like—just be sure that you expose an environment variable called
`ORACLE_HOME` that points to your installation.

### 3. Build oracle_fdw

Download the [latest release](https://github.com/laurenz/oracle_fdw/releases) of oracle_fdw
and unpack it. `cd` into the unpacked directory and compile the extension with `make`.

Once oracle_fdw is compiled, you have two options for proceeding: [modify the compiled binary](#option-1-alter-the-oraclefdw-binary)
or [symlink your instantclient](#option-2-symlink-libclntshdylib111-to-an-expected-location) to a location it can find. Without doing one of these, you'll
likely encounter this error upon loading the extension in Postgres:

```
ERROR:  could not load library "/usr/local/lib/postgresql/oracle_fdw.so":
  dlopen(/usr/local/lib/postgresql/oracle_fdw.so, 10):
  Library not loaded: @rpath/libclntsh.dylib.11.1
  Referenced from: /usr/local/lib/postgresql/oracle_fdw.so
  Reason: image not found
```

#### Option 1: alter the oracle_fdw binary

This is the option I chose. It lets me tweak the compiled binary while leaving everything
else as-is. After compiling with `make` (and before `make install`), issue this command:

```
install_name_tool -add_rpath $ORACLE_HOME oracle_fdw.so
```

This adds your `ORACLE_HOME` path to the list of `rpath`s in `oracle_fdw.so`.

#### Option 2: symlink libclntsh.dylib.11.1 to an expected location

The other option is to symlink `libclntsh.dylib.11.1` to one of the default
search paths, `~/lib`:

```
ln -s $ORACLE_HOME/libclntsh.dylib.11.1 ~/lib/
```

### 4. Install oracle_fdw

Now that oracle_fdw is compiled and ready to link up the instantclient, install
it and its friends with `make install`.

At this point you should be able to fire up your favorite Postgres client and enable the
extension with this SQL:

```
CREATE EXTENSION oracle_fdw;
```

If you don't see any gnarly errors, you're in business! I'll write up another post
soon that demonstrates how to use the extension.


