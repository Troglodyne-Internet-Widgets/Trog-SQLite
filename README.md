# NAME

Trog::SQLite - An SQLite handle with its schema applied, its updates run, and its pragmas set

# VERSION

version 0.005

# SYNOPSIS

```perl
use Trog::SQLite;

my $dbh = Trog::SQLite::dbh( 'schema/app.sql', '/var/lib/app/app.db' );

Trog::SQLite::bulk_insert( $dbh, 'tags', [qw{post tag}], 'IGNORE', 1, 'perl', 1, 'sqlite' );
```

# DESCRIPTION

`dbh` opens an SQLite database, applies its schema, runs the updates of that
schema that it has not run yet, and sets the pragmas that let several processes
share one database.  It caches the handle, so every later call for that file
gets the same one.

Everything here dies when something goes wrong.

## Schema updates

A schema file holds the first version of the schema, version 0.  Write each
statement in it so that it can run again, such as `CREATE TABLE IF NOT EXISTS`,
because `dbh` applies it on every new handle.

A change to the schema after the first version goes in an update file beside
it.  Each update has a number, from 1 up with no gaps, and its name is the name
of the schema with that number:

```
schema/app.sql              version 0
schema/app.v1.update.sql    version 1
schema/app.v2.update.sql    version 2
```

Each database records its version in the table `trog_sqlite_schema_version`.
`dbh` makes that table in every database that it opens, with one row and a
version of 0.  Then it runs each update with a number above that version, in
order.  Each update runs in one `BEGIN IMMEDIATE` transaction, together with
the change to the recorded version.  So an update that fails changes nothing,
and the version stays where it was.  When two processes open the database at
the same time, the second one waits for the first, and then finds the update
done.

A new database gets the schema, and then every update.  So the update files
together with the schema describe the current tables, and nobody needs to
change the schema file after its first release.

An update file must not hold `BEGIN` or `COMMIT`, because it already runs in
a transaction.  Foreign keys are not enforced while the updates run, so an
update can rebuild a table that other tables refer to.  After each update,
`PRAGMA foreign_key_check` must report nothing, or the update is rolled back.

# FUNCTIONS

## dbh($schema, $dbname, \\%options)

Returns a handle to the database in the file `$dbname`, which is made if it
does not exist.  If you give `$schema`, its DDL and its updates are applied
first, as ["Schema updates"](#schema-updates) says.  Omit `$schema` for a database that already
has its schema.

`%options` is optional.  It has one key:

- `integrity_check`: when true, run `integrity_check` on the database
before the schema, and die if the check fails.  The check reads every page, so
it takes as long as the database is large, and it is off unless you ask.

Dies if `\%options` is not a hash reference, or has a key that is not listed
here.

The handle has `RaiseError` on, and these pragmas:

- `busy_timeout = 5000`, so that a second process waits for five seconds
for a lock rather than failing at once with `database is locked`.
- `journal_mode = WAL` and `synchronous = NORMAL`, so that readers do
not block a writer.  SQLite does not wait for a lock to change the journal
mode, so `wal` tries again for as long as `busy_timeout` waits.
- `foreign_keys = ON`, set after the updates.

The handle is cached for the rest of the process.  A second call for the same
`$dbname` returns the cached handle, and does not apply `$schema` again or
check the database again.  A handle that fails its check is not cached.

Do not call this during `BEGIN` or outside a sub.

A handle must not outlive a fork.  The cache remembers the process that it
belongs to, so a forked child that calls this gets a new handle of its own,
and the parent keeps its handle.  The child does not need to call `forget`.

The handle has `AutoInactiveDestroy` on, so a forked child that drops it, or
exits, does not close it.  Otherwise [DBI](https://metacpan.org/pod/DBI) would roll back an open transaction of
the parent from the child, and SQLite can lose the pages of that transaction
that it already wrote to the WAL.

## wal($db)

Sets the journal mode of the handle `$db` to WAL, and returns 1.  When another
connection holds a write lock, SQLite fails the change at once with
`SQLITE_BUSY` and does not wait for `busy_timeout`.  So this tries again
every `$WAL_DELAY` seconds, for as long as `$BUSY_TIMEOUT` says.  Dies with
the last error when the lock outlasts that, and at once on any other error.

## integrity\_check($db)

Runs `PRAGMA integrity_check` on the handle `$db`, and returns 1 if it
reports `ok`.  Otherwise dies with the name of the file and every problem that
the check reported, one on each line.

## apply($db, $file)

Runs every statement in the SQL file `$file` on the handle `$db`, and returns
1\.  Dies if the file cannot be read, or if a statement fails.

## @updates = updates($schema)

Returns the update files of the schema file `$schema`, as a list of
`[ $version, $path ]` pairs in the order of their versions.  See
["Schema updates"](#schema-updates).

Dies if the versions have a gap, such as a `v3` with no `v2`, because then one
of the files is misnamed or missing, and running the others would skip it.

## $version = schema\_version($db)

Returns the version of the schema that the database of `$db` records.  It makes
the table that records it, with one row and a version of 0, if the database has
none.

## $version = update($db, $schema)

Runs each update of `$schema` that the database of `$db` has not run, as
["Schema updates"](#schema-updates) says, and returns the version of the database afterwards.
`dbh` calls this.  Dies with the name of the file if an update fails, and that
update then changes nothing.

## forget()

Drops the cached handles and returns 1, so that the next `dbh` for each file
opens a new handle.  `dbh` calls this itself in a forked child.

## transaction($db, $work)

Runs the sub `$work` in one transaction on the handle `$db`, and returns 1.
If `$work` dies, the transaction is rolled back, and this dies with the error
of `$work`.

If `$db` is already in a transaction, `$work` runs in that one, and the code
that began it commits it or rolls it back.  So a sub that needs a transaction
can call this whether or not its caller began one.

## bulk\_insert($dbh, $table, \\@keys, $action, @values)

Inserts `@values` into the columns `@keys` of `$table`, and returns 1.
`@values` is one row after another, each with one value for each key:

```perl
my @keys   = qw{A B C};
my @values = qw{1 2 3 4 5 6 7 8 9};    # three rows
```

`$action` is the conflict clause of the `INSERT`: one of `ROLLBACK`,
`ABORT`, `FAIL`, `IGNORE` or `REPLACE`.  With foreign keys off, `REPLACE`
inserts a row or replaces the one with its key, with less code than
`ON CONFLICT ... DO UPDATE`.

One statement takes as many parameters as the handle allows, which is 999 in
older builds of SQLite and 32766 in newer ones.  So the rows go in as many
statements as that needs, which reuse one prepared statement.  Split a very
large `@values` into batches that fit in memory.

Dies if `$action` is not one of those, if `@keys` is empty, or if the number
of values is not a multiple of the number of keys.

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/Troglodyne-Internet-Widgets/Trog-SQLite/issues](https://github.com/Troglodyne-Internet-Widgets/Trog-SQLite/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# CONTRIBUTOR

troglodyne-bot <morpheus@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
