---
layout: post
title: "Getting Clojure/Leiningen to talk to Oracle"
lead: "How to build a Clojure/Leiningen project that uses Oracle ojdbc6.jar without
getting the dreaded `SQLException: No suitable driver found for jdbc:oracle:thin:...` error."
---
I've just started learning Clojure and have been trying desperately to
get my Leiningen project to talk to an Oracle database. All of my
enterprise's data is wrapped up in Oracle, so to do anyting useful or
interesting with Clojure, I had to first overcome this first step.

The following is surely a no-brainer for experienced Java developers, but I come from
Ruby and am new to both Java *and* Clojure. I hope this helps others
coming into Clojure from this angle.

## Download Oracle JDBC driver

If you don't already have an Oracle JDBC driver (ojdbc5.jar, ojdbc6.jar,
ojdbc14.jar), you'll want to [download it from Oracle](http://www.oracle.com/technetwork/database/enterprise-edition/jdbc-112010-090769.html) first.

## Install ojdbc jar to local Maven repository

Install your JDBC driver to your local Maven repo with this command:

{% highlight bash %}
mvn install:install-file -X -DgroupId=local -DartifactId=ojdbc6 -Dversion=11.2.0.3 -Dpackaging=jar -Dfile=/opt/oracle/instantclient/ojdbc6.jar -DgeneratePom=true
{% endhighlight %}

Note a few things here:

- `groupId`, which would normally be something
like `com.oracle` is replaced with `local`. This can be anything you
want; `self` or `poo` would work just as well.
- Make sure `version` matches the version you downloaded from Oracle.
- `file` needs to point to an absolute path to your jar file. Relative
  paths don't seem to work.

## Reference your local respository in project.clj

In your project.clj file, you can now specify your proprietary library
as you would any other dependency. Just be sure to prefix it with the
value you used for `groupId` in your maven command; `local` in this case:

{% highlight clojure %}
  :dependencies [[org.clojure/clojure "1.5.1"]
                 [org.clojure/java.jdbc "0.3.0-alpha4"]
                 [local/ojdbc6 "11.2.0.3"]
{% endhighlight %}

## Trying it out

Huzzah! My cheesy query now returns results:

{% highlight clojure %}
(require '[clojure.java.jdbc :as j]
         '[clojure.java.jdbc.sql :as s])

(def db {:classname "oracle.jdbc.driver.OracleDriver"
         :subprotocol "oracle"
         :subname "thin:@somehost:1521/some_service"
         :user "some_user"
         :password "some_pass"})

(j/query db (s/select * :dual))
{% endhighlight %}

Returns the familiar ol':

{% highlight clojure %}
({:dummy "X"})
{% endhighlight %}

