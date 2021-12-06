libpqxx
=======

Welcome to libpqxx, the C++ API to the PostgreSQL database management system.

Home page: http://pqxx.org/development/libpqxx/

Find libpqxx on Github: https://github.com/jtv/libpqxx

Documentation on Read The Docs: https://readthedocs.org/projects/libpqxx/

Compiling this package requires PostgreSQL to be installed -- or at least the C
headers and library for client development.  The library builds on top of
PostgreSQL's standard C API, libpq, though your code won't notice.

If you're getting the code straight from the Git repo, the head of the `master`
branch represents the current _development version._  Releases are tags on
commits in `master`.  For example, to get version 7.1.1:

    git checkout 7.1.1


Upgrade notes
-------------

**The 7.x versions require C++17.**  Make sure your compiler is up to date.
For libpqxx 8.x you will need C++20.

Also, **7.0 makes some breaking changes in rarely used APIs:**
* There is just a single `connection` class.  It connects immediately.
* Custom `connection` classes are no longer supported.
* It's no longer possible to reactivate a connection once it's been closed.
* The API for defining string conversions has changed.

If you're defining your own type conversions, **7.1 requires one additional
field in your `nullness` traits.**


Building libpqxx
----------------

There are two different ways of building libpqxx from the command line:
1. Using CMake, on any system which supports it.
2. On Unix-like systems, using a `configure` script.

"Unix-like" systems include GNU/Linux, Apple macOS and the BSD family, AIX,
HP-UX, Irix, Solaris, etc.  Even on Microsoft Windows, a Unix-like environment
such as WSL, Cygwin, or MinGW should work.

You'll find detailed build and install instructions in `BUILDING-configure.md`
and `BUILDING-cmake.md`, respectively.

And if you're working with Microsoft Visual Studio, have a look at Gordon
Elliott's _Easy-PQXX Build for Windows Visual Studio_ project:

    https://github.com/GordonLElliott/Easy-PQXX-Build-for-Windows-Visual-Studio

At the time of writing, June 2020, this is still in development and in need of
testers and feedback.


Documentation
-------------

Building the library, if you have the right tools installed, generates HTML
documentation in the `doc/` directory.  It is based on the headers in
`include/pqxx/` and text in `include/pqxx/doc/`.  The documentation is also
available online at readthedocs.org.


Programming with libpqxx
------------------------

How do you talk to the database?  There's a pattern.

First you connect to the database by creating a `pqxx::connection` object.  See
the `pqxx/connection.hxx` header for an overview of what it can do, but most of
the time you just create it at the beginning and destroy it when you're done.

Then, to "do actual stuff" on the database, you create a transaction object.
The usual type for that is `pqxx::work`, which is a convenience alias for the
`pqxx::transaction` class template.  For an overview, look at its parent class,
`pqxx::transaction_base` in `pqxx/transaction_base.hxx`.

Within the transaction, you can interact with the database as much as you like.
Usually you'll do that through member functions on the transaction object.  For
example, there's a family of functions with names like `exec()` to execute SQL
commands.

When you've completed a unit of work, you _commit_ the transaction.  Or if you
don't want to, you can either explicitly _abort_ it, or just destroy the
transaction — same effect.  Just make sure you always destroy the transaction
object _before the connection_ object.  Think of the transaction as a "nested"
object that lives inside the scope of the connection.

After you've committed, aborted, or destroyed the transaction, feel free to
start a new one if you want to do more work on the same connection.  And of
course you can have multiple connections at the same time.  A connection does
not care what another connection is doing.

By the way those `*.hxx` headers are not the ones you include in your code.
Instead, include the versions without filename suffix, i.e. `pqxx/connection`
etc.  Those will include the actual .hxx files for you.

Here's a simple example program to get you going, with full error handling:

```c++
    #include <iostream>
    #include <pqxx/pqxx>

    int main()
    {
        try
        {
            // Connect to the database.
            pqxx::connection conn;
            std::cout << "Connected to " << conn.dbname() << '\n';

            // Start a transaction.
            pqxx::work tx{conn};

            // Perform a query and retrieve all results.
            pqxx::result R{tx.exec("SELECT name FROM employee")};

            // Iterate over results.
            std::cout << "Found " << R.size() << "employees:\n";
            for (auto row: R)
                std::cout << row[0].c_str() << '\n';

            // Perform a query and check that it returns no result.
            std::cout << "Doubling all employees' salaries...\n";
            tx.exec0("UPDATE employee SET salary = salary*2");

            // Commit the transaction.
            std::cout << "Making changes definite: ";
            tx.commit();
            std::cout << "OK.\n";
        }
        catch (std::exception const &e)
        {
            std::cerr << e.what() << '\n';
            return 1;
        }
        return 0;
    }
```

That example also shows you the `pqxx::result` class.  It represents a complete
query result, which libpqxx receives from the database as one block.  A
`result` is a container of `row`s, and each `row` is a container of `field`s.

You can read a field in the text format in which the database sends it over, or
you can convert it to a type of your choice — assuming that the text represents
a valid item of that type.

There are many options for doing all this: you can index the result by row
number and then index the row by column name.  Or you can index the row by
column number.  Or you can just index the result by row and then convert the
whole row to C++ types at once, with separate types per column.

If you need it really fast, you can also _stream_ data from most queries.  That
way you can start processing before all the data have come in.  This method
doesn't use a `result` at all.  It just reads one row at a time and converts it
to a `std::tuple` of the column types you select.

```c++
    #include <iostream>
    #include <pqxx/pqxx>

    int main()
    {
        try
        {
            pqxx::connection conn;
            pqxx::work tx{conn};

            // Perform a query and stream the results.
            for (
              auto [name, salary] :
              tx.stream<std::string_view, double>(
                "SELECT name, salary FROM employee")
            )
            {
              // Each iteration is one result row.
              // The "name" is the employee's name.  We asked for it as a
              // std::string_view.  It will only be valid for this one
              // iteration, so we can't access its data afterwards.
              std::cout << name << '\t' << salary << '\n';
            }
        }
        catch (std::exception const &e)
        {
            std::cerr << e.what() << '\n';
            return 1;
        }
        return 0;
    }
```


Connection strings
------------------

When you create your connection, you can tell it where to connect and how.  You
do that by passing a _connection string_ to the `pqxx::connection` constructor.

Postgres connection strings state which database server you wish to connect to,
under which username, using which password, and so on.  Their format is defined
in the documentation for libpq, the C client interface for PostgreSQL.
Alternatively, these values may be defined by setting certain environment
variables as documented in e.g. the manual for psql, the command line interface
to PostgreSQL.  Again the definitions are the same for libpqxx-based programs.

The connection strings and variables are not fully and definitively documented
here; this document will tell you just enough to get going.  Check the
PostgreSQL documentation for authoritative information.

The connection string consists of attribute=value pairs separated by spaces,
e.g. "user=john password=1x2y3z4".  The valid attributes include:
* `host` —
  Name of server to connect to, or the full file path (beginning with a
  slash) to a Unix-domain socket on the local machine.  Defaults to
  "/tmp".  Equivalent to (but overrides) environment variable PGHOST.
* `hostaddr` —
  IP address of a server to connect to; mutually exclusive with "host".
* `port` —
  Port number at the server host to connect to, or socket file name
  extension for Unix-domain connections.  Equivalent to (but overrides)
  environment variable PGPORT.
* `dbname` —
  Name of the database to connect to.  A single server may host multiple
  databases.  Defaults to the same name as the current user's name.
  Equivalent to (but overrides) environment variable PGDATABASE.
* `user` —
  User name to connect under.  This defaults to the name of the current
  user, although PostgreSQL users are not necessarily the same thing as
  system users.
* `requiressl` —
  If set to 1, demands an encrypted SSL connection (and fails if no SSL
  connection can be created).

Settings in the connection strings override the environment variables, which in
turn override the default, on a variable-by-variable basis.  You only need to
define those variables that require non-default values.


Linking with libpqxx
--------------------

To link your final program, make sure you link to both the C-level libpq
library and the actual C++ library, libpqxx.  With most Unix-style compilers,
you'd do this using the options

    -lpqxx -lpq

while linking.  Both libraries must be in your link path, so the linker knows
where to find them.  Any dynamic libraries you use must also be in a place
where the loader can find them when loading your program at runtime.

Some users have reported problems using the above syntax, however, particularly
when multiple versions of libpqxx are partially or incorrectly installed on the
system.  If you get massive link errors, try removing the "-lpqxx" argument
from the command line and replacing it with the name of the libpqxx library
binary instead.  That's typically libpqxx.a, but you'll have to add the path to
its location as well, e.g. /usr/local/pqxx/lib/libpqxx.a.  This will ensure
that the linker will use that exact version of the library rather than one
found elsewhere on the system, and eliminate worries about the exact right
version of the library being installed with your program..
