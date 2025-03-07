Tutorial
========

This documentation section for pgcopydb contains a list of classic pgcopydb
use-cases. For details about the commands and their options see the manual
page for each command at :ref:`pgcopydb`.

Copy Postgres Database to a new server
--------------------------------------

The simplest way to use pgcopydb is to just use the :ref:`pgcopydb_clone`
command as in the following example.

::
   
   $ export PGCOPYDB_SOURCE_PGURI="dbname=pagila"
   $ export PGCOPYDB_TARGET_PGURI="postgres://user@target:5432/pagila"

   $ pgcopydb clone

Note that the options ``--source`` and ``--target`` can also be used to set
the Postgres connection strings to the databases. Using environment
variables is particulary useful when using Docker containers.

You might also notice here that both the source and target Postgres
databases must already exist for pgcopydb to operate.

Copy Postgres users and extensions
----------------------------------

To copy Postgres users a privileged connection to the target database must
be made, and to include passwords a privileged connection to the source
database must be made too. It is sometimes wanted to limit these privileged
connection to the minimum, and then an approach that looks like the
following may be used:

::

   $ coproc ( pgcopydb snapshot --source ... )

   # first two commands would use a superuser role
   $ pgcopydb copy roles --source ... --target ...
   $ pgcopydb copy extensions --source ... --target ...

   # now it's possible to use a non-superuser role
   $ pgcopydb clone --skip-extensions --source ... --target ...

   $ kill -TERM ${COPROC_PID}
   $ wait ${COPROC_PID}


How to edit the schema when copying a database?
-----------------------------------------------

It is possible to split pgcopydb operations and run them one at a time.


Please note that concurrency and performance characteristics that depend on
concurrency are then going to be pretty limited compared to the main
``pgcopydb clone`` command where different sections are running concurrently
to one-another.

Still running operations with more control over the different steps can be
necessary. An interesting use-case consists of injecting schema changes
before copying the data over:

::
   
   #
   # pgcopydb uses the environment variables
   #
   $ export PGCOPYDB_SOURCE_PGURI=...
   $ export PGCOPYDB_TARGET_PGURI=...
   
   #
   # we need to export a snapshot, and keep it while the indivual steps are
   # running, one at a time
   #
   $ coproc ( pgcopydb snapshot )
   
   $ pgcopydb dump schema --resume
   $ pgcopydb restore pre-data --resume

   #
   # Here you can implement your own SQL commands on the target database.
   #
   $ psql -d ${PGCOPYDB_TARGET_PGURI} -f schema-changes.sql

   # Now get back to copying the table-data, indexes, constraints, sequences
   $ pgcopydb copy data --resume   
   $ pgcopydb restore post-data --resume
   
   $ kill -TERM ${COPROC_PID}
   $ wait ${COPROC_PID}

   $ pgcopydb list progress --summary
      
Note that to ensure consistency of operations, the ``pgcopydb snapshot``
command has been used. See :ref:`resuming_operations` for details.

Follow mode, or Change Data Capture
-----------------------------------

When implementing Change Data Capture then more sync points are needed
between pgcopydb and the application in order to implement a clean cutover.

Start with the initial copy and the replication setup:

::
   
   $ export PGCOPYDB_SOURCE_PGURI="dbname=pagila"
   $ export PGCOPYDB_TARGET_PGURI="postgres://user@target:5432/pagila"

   $ pgcopydb clone --follow

While the command is running, check the replication progress made by
pgcopydb with the Postgres `pg_stat_replication`_ view.

__ https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW

When the lag is close enough for your maintenance window specifications,
then it's time to disconnect applications from the source database, finish
the migration off, and re-connect your applications to the target database:

::

   $ pgcopydb stream sentinel set endpos --current

This command must run with the same ``--dir`` as the main ``pgcopydb clone
--follow`` command, in order to share the same internal catalogs with the
running processes.

When the migration is over with, now cleanup the resources created for the
Change Data Capture with the following command:

::

   $ pgcopydb stream cleanup

See also :ref:`change_data_capture` for mode details and other modes of
operations.
   
How to validate schema and data migration?
------------------------------------------

The command :ref:`pgcopydb_compare_schema` is limited to comparing the
metadata that pgcopydb grabs about the Postgres schema at the moment. This
means comparing the list of tables, their attributes, their indexes and
constraints, and the sequences values.

The command :ref:`pgcopydb_compare_data` run a SQL query that computes a
checksum of the data on each Postgres instance, for each table, and then
only compares there checksums. This is not a full comparison of the data
set, cases where the checksum are the same and the data differ can be found.

::

   $ pgcopydb compare schema
   $ pgcopydb compare data
