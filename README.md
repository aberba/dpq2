dpq2
====
[![Build Status](https://travis-ci.org/denizzzka/dpq2.svg?branch=master)](https://travis-ci.org/denizzzka/dpq2)

This is yet another attempt to create a good interface to PostgreSQL for the 
D programming language.

It adds only tiny overhead to the original low level library libpq but
make convenient use PostgreSQL from D.


Features
--------

* Arguments list support
* Async queries support
* Reading of the text query results to native D text types
* Representation of the binary query results to native D types
 * Text types
 * Integer and decimal types (except "numeric")
 * Timestamp type (with and without timezone)
* Access to PostgreSQL's multidimensional arrays
* LISTEN/NOTIFY support

Building
--------

Bindings for libpq can be static or dynamic.

The static bindings are generated by default. Add `--config=dynamic`
to the `dub` parameters to generate dynamic bindings.

Example
-------
```D
#!/usr/bin/env rdmd

import dpq2.all;
import std.stdio: writeln;

void main()
{
    Connection conn = new Connection;
    conn.connString = "dbname=postgres";
    conn.connect();

    // Only text query result can be obtained by this call:
    auto s = conn.exec(
        "SELECT now() as current_time, 'abc'::text as field_name, "
        "123 as field_3, 456.78 as field_4"
        );
    
    writeln( "Text query result: ", s[0][3].as!PGtext );
    
    // Separated arguments query with binary result:
    queryParams p;
    p.sqlCommand = "SELECT "
        "$1::double precision as double_field, "
        "$2::timestamp with time zone as time_field, "
        "$3::text, "
        "$4::text as null_field, "
        "array['first', 'second', NULL]::text[] as array_field, "
        "$5::integer[] as multi_array";
    
    p.args.length = 5;
    
    p.args[0].value = "-1234.56789012345";
    p.args[1].value = "2012-10-04 11:00:21.227803+08";
    p.args[2].value = "first line\nsecond line";
    p.args[3].value = null;
    p.args[4].value = "{{1, 2, 3}, {4, 5, 6}}";
    
    auto r = conn.exec(p);
    
    writeln( "0: ", r[0]["double_field"].as!PGdouble_precision );
    writeln( "1: ", r[0]["time_field"].as!PGtime_stamp.toSimpleString );
    writeln( "2: ", r[0][2].as!PGtext );
    writeln( "3.1 isNull: ", r[0][3].isNull );
    writeln( "3.2 isNULL: ", r[0].isNULL(3) );
    writeln( "4.1: ", r[0][4].asArray[0].as!PGtext );
    writeln( "4.2: ", r[0][4].asArray[1].as!PGtext );
    writeln( "4.3: ", r[0]["array_field"].asArray[2].isNull );
    writeln( "4.4: ", r[0]["array_field"].asArray.isNULL(2) );
    writeln( "5: ", r[0]["multi_array"].asArray.getValue(1, 2).as!PGinteger );
    
    version(LDC) delete r; // before Derelict unloads its bindings (prevents SIGSEGV)
}
```
####Compile and run:
```
$ cd example
$ dub build --build=release
$ sudo -u postgres ./dpq2-example
Text query result: 456.78
0: -1234.57
1: 0013-Oct-05 03:00:21.227803Z
2: first line
second line
3.1 isNull: true
3.2 isNULL: true
4.1: first
4.2: second
4.3: true
4.4: true
5: 6
```

TODO
----

* PostGIS binary data support
* Checking types by Oid
* Row by row result reading
* Binary arguments types
