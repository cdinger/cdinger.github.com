---
layout: post
title: Using erloci to talk to Oracle in Elixir
lead: A handful of examples of interacting with an Oracle data from Elixir via the erloci Erlang library.
---

One of the great things about Elixir is its clean Erlang interoperability. The
adoption rate of new languages like Elixir depends greatly on the ecosystem of
available libraries. While the Elixir ecosystem is growing, being able to lean on the
twenty(ish) years of existing open source Erlang libraries helps a lot.

Here are some examples of interacting with an Oracle database from Elixir via the
[erloci](https://github.com/K2InformaticsGmbH/erloci) Erlang library.

### Connect

`erloci` requires you to specify connection details as a TNS string. This can be
slightly annoying but you can easily interpolate host, port, and service name into
a TNS-style format.

```elixir
{% raw %}
# connect
host = "somehost.yourorg.com"
port = "1521"
service = "someservice"
username = "username"
password = "password"
tns = "(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=tcp)(HOST=#{host})(PORT=#{port})))(CONNECT_DATA=(SERVICE_NAME=#{service})))"
oci = :erloci.new([])
session = oci.get_session(tns, username, password)

# DDL
create = session.prep_sql("create table blahs (id integer, name varchar(100))")
{:executed, 0} = create.exec_stmt
{% endraw %}
```

### Select

All SQL interactions with `erloci` are proper prepared statements. The `exec_stmt/0` function
returns a list of column and type information. Data must be explicitly fetched with `fetch_rows/1`.

```elixir
{% raw %}
iex> sql = session.prep_sql("select * from blahs")
iex> {:cols, columns} = sql.exec_stmt
{:cols, [{"ID", :SQLT_NUM, 22, 38, 0}, {"NAME", :SQLT_CHR, 200, 0, 0}]}
iex> {{:rows, rows}, false} = sql.fetch_rows(1)
{{:rows,
  [[<<2, 193, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0>>,
    "ahoy!"]]}, false}
{% endraw %}
```

What the heck is `<<2, 193, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0>>`?
Well, from the column information returned from `exec_stmt` we know it's of type `:SQLT_NUM`
but what `erloci` is returning is binary. You'll want to convert it using `:oci_util.oranumber_decode/1`.
In this case, it's the integer `2`. See the [oci_util](https://github.com/K2InformaticsGmbH/erloci/blob/master/src/oci_util.erl)
module for a pile of other handy conversion function and utilities.

### Insert

`erloci` has full support for bound parameters. Parameters that are to be bound can
be named (like `:id` or `:name`) or can use numbers (`:1`, `:2`, `:3`) in the SQL statement.
`bind_vars/1` must then be called to supply type information for each of the bound parameters.
Finally, `exec_stmt/1` is used to pass in values and execute the insert.

```elixir
{% raw %}
# insert with bound parameters
insert = session.prep_sql("insert into blahs (id, name) values (:id, :name)")
:ok = insert.bind_vars([{":id", :SQLT_INT}, {":name", :SQLT_CHR}])
{:rowids, ["AABUUnAQAAAEGgTAAA"]} = insert.exec_stmt([{1, "blahblah"}])
{% endraw %}
```

### Errors

When errors are encountered `erloci` returns the Oracle error code and message.

```elixir
{% raw %}
create = session.prep_sql("create table blahs (id integer, name varchar(100))")
{:error, {955, "ORA-00955: name is already used by an existing object\n"}} = create.exec_stmt
{% endraw %}
```

