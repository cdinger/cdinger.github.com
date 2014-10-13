---
layout: post
title: "Query for primary keys on a PeopleSoft table"
lead: "PeopleSoft tables don't use real primary keys—they use logical primary keys. So if you want to
       find the keys for say, the acad_prog_tbl table, one usually has to resort to venturing into the
       PeopleSoft web application (or worse, App Designer). If you want to save yourself this pain, you
       can query these keys from one of PeopleSoft's setup tables, pskeydefn."
---

My day job has be building Rails apps that sling data to and from PeopleSoft
instances. Because of this, I often find myself looking up a particular table's primary key.
If you ever had to do this with a PeopleSoft table, you know it's a pain. PeopleSoft doesn't use
real DB keys, so to look these up you have to ask PeopleSoft itself. For me, this always involved logging 
into the web application and fiddling with the reporting engine or worse, firing up a VM with App Designer.

I've just stumbled upon the table where PeopleSoft stores this information, `pskeydefn`. So to query the
primary keys for say, ps_acad_prog_tbl, query like so:

{% highlight clojure %}
SELECT fieldname
FROM pskeydefn
WHERE recname='ACAD_PROG_TBL' -- or any PS table
  and indexid='_'
ORDER BY keyposn
{% endhighlight %}

This is huge time saver for me. If anyone else lives in this strange world between web apps and PS, I hope this is useful.

