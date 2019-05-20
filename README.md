Project Migrations
==================

migrate-proj is a tool for maintaining a large set of git repositories. It makes it easy to apply common updates to many projects. You can also use migrate-proj to update an old branch or commit so that the older branch or commit can be rebuilt with the latest build modifications.

In the world of Ruby on Rails, databases are kept up-to-date with the latest
version of the schema by applying a set of "database migrations." This has
proven to be a powerful and effective way of incrementally improving a database
over time.

Project migrations are similar to database migrations, except that they operate
against a source code project. Project migrations can be used to keep all of your
projects in-sync with the latest layout, configuration, and build
modifications.

Benefits
--------
Project migrations are ideal when one or more updates may need to be applied
repeatedly. The most common case is applying the same updates to many
projects, but they also have uses within a single project. Some other
scenarios in which project migrations can be useful include:

* applying updates to many branches
* applying updates to a forgotten or stale branch
* applying updates to an old commit so that you can build it

Most "migrations" are updates to build scripts or build configuration files but
they can also be for other types of changes as well. For example, you might add
a migration which adds entries to a .gitignore file or restructures the project
layout.

Migrating a project
-------------------

To update a project to the latest configuration, run the `migrate_proj`
command from the root of the project. All of the migrations which have not
been previously applied will be executed.

Each project contains a `.migraiton-level` file which the `migrate_proj`
command uses to determine which migrations have already been applied to a
project. If `migrate_proj` is run within a project that does not yet have
a `.migration-level` file, then all of the migrations are applied and a new
`.migrate-level` file is created.

Once each migration has been applied, the `migrate_proj` command
automatically commits the changes (currently, only Git is supported).

Writing a new migration
-----------------------
To add a new migration:

1. Within the migrations directory, create a new directory following the
   migration directory naming rules (see below).
2. Create a `description.txt` file in the new directory
3. Create an executable `do_migration` script in the new directory
4. Optionally write variables to standard out during execution of
   `do_migration` to render variables through your `description.txt`

That's it! Your new migration will be applied the next time `migrate_proj` is
run against a project.

### Where to put migrations

By default, migrate_proj looks for migrations in the migrations subdirectory of
the directory the script lives in. The default migrations directory contains a
.gitignore file that ignores all files within the directory to prevent
accidentally pushing a migration up to the project itself. If you want to
use git to manage your migrations (and you probably do), then you will want to
change the default location.

You can change the directory migrate_proj looks for migrations in by either setting the
PROJ_MIGRATIONS_HOME environment variable or by passing the `--migrations-home`
command line flag.

### Migration directories

Each migration is stored in a directory with the following naming syntax:

    LEVEL["-ALWAYS"][NAME]

`LEVEL` is the only required component of the directory name, and must be a
positive integer. `LEVEL` may be zero-padded, and by convention is zero-padded
to three digits (so that the migrations sort nicely when using `ls`). `LEVEL`
defines the order the migrations run in and is also used by the `migrate_proj`
script to determine which migrations have not yet been applied to the
project.

The `migrate_proj` command does not enforce `LEVEL` to be unique, but you
really should use unique values. The order in which two migrations with the
same `LEVEL` run is undefined. So don't do it. Always use unique values.

If `LEVEL` is immediately followed by "-ALWAYS" (without the quotes), then the
migration is applied any time other migrations are applied, regardless of the
project's level. Such migrations are known as "Always Run" migrations. Always
Run migrations MUST be idempotent since they will be run more than once.

`NAME` is can be any arbitrary text that summarizes what the
migration does. The migrate_proj command prints the name when the --list
argument is used and when executing migrations. When printing a migration's
name, migrate_proj replaces underscores with spaces. This allows you to avoid
putting spaces in your directory names.

Although `NAME` is optional, you really should include one because it makes it
easier to understand which directory contains which migration, and it makes for
nicer command output.

Examples:

* `001-ALWAYS-Update_copyright_headers` -- the first migration to execute, and
  is always executed.
* `123-Replace_monkeys_with_kittens` -- runs after any migration with a level
  less than 123, but only executes once per project.

Each migration directory contains at least two files:

* `do_migration` - the script that actually performs the migration.

* `description.txt` - contains the textual description of the migration. The text contained in this file is used in the
commit message that `migrate_proj` generates.

### do_migration

Each migration directory must contain an executable `do_migration` script. The
script may be written in any scripting language that is available on the
system.

Other files may also be in the migration directory. Thus, you don't have to put
everything in the `do_migration` script, but can break things up into multiple
files.

Any output from do_migration that is sent to standard out or standard error is
not displayed to the user, unless migrate_proj is called with the --verbose
flag, or if the script returns a non-zero exit code. If a non-zero exit code is
returned, then migrate_proj assumes that the migration failed and outputs the
standard out and standard error to help troubleshoot the issue.

#### Adding new files

If your `do_migration` script adds a new file, be sure to call `git add
NEWFILE` somewhere in your script. When `migrate_proj` creates the commit, it
does NOT automatically add new files, so you need to make sure your
`do_migration` script does.

### description.txt

Each migration directory must contain a `description.txt` file. The content of
the `description.txt` is used in the commit messages generated by
the `migrate_proj` script.

`migrate_proj` is not yet smart enough to auto-wrap the text within a commit
message. However, it does preserve any newlines contained within
description.txt. As such, you should wrap the text within `description.txt` so
that git commit message line length conventions are honored:

  - Subject line should be no more than 50 characters
  - Body should wrap at 72 characters

#### Using properties in description.txt

You can include properties (variables) in description.txt that will be replaced with a value
generated by the do_migration script when migrate_proj creates the commit
message for the migration. To include a property, use a dollar sign (\$) before
the property name, or surround the property name with \${}.

To generate values for the properties, update your do_migration script to output
the property values to standard out at the very end of the script. The values
of the properties should be preceded by the string `--@Properties--` and a
newline. After the `--@Properties--` line, provide the values for the
properties using the [Java Properties File
Format](https://en.wikipedia.org/wiki/.properties#Format).

The do_migration script must provide values for every property in the
description.txt file. If a value for a variable is not found, then the
migration will fail.

Even though they are written to standard out, the `--@Properties--` line and
everything that follows it is never displayed to the user.  Thus, make sure any
diagnostic information is written to standard out before writing out the
property values.

Here is an example of how to use properties in your commit message templates. If
we have a description.txt file that contains the text...

    Upgrades the astro spanner to version ${version}

    The new version was created on $creation_date.

...then our do_migration script should output something like:

    Successfully update the astro spanner version!

    --@Properties--
    version=1.2.22
    creation_date=2016-02-21

When migrate_proj generates a commit message for the migration, it will
generate:

    Upgrades the astro spanner to version 1.2.22

    The new version was created on 2016-02-21.

CONTRIBUTING
============
Contributions are welcome! See [CONRIBUTING.md](CONTRIBUTING.md) for details.

License
=======

This project is licensed under [Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0).



