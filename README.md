Now that exclusive backup with `pg_start_backup()` and `pg_stop_backup()`
has been deprecated, it is more difficult to perform a PostgreSQL online
file system backup using a pre-backup and a post-backup script.

If you are reading this, please yell at the community for deprecating this
feature -- or by improving the non-exclusive mode to be operable across
sessions. (It would require pg_stop_backup() to take a label as a parameter,
but other than that, would be achievable).

This project provies a single, sourceable script and configuration file 
which you can incorporate into your backup script(s), using postgresql's
["non-exclusive" backup scheme](https://www.postgresql.org/docs/current/continuous-archiving.html#BACKUP-LOWLEVEL-BASE-BACKUP-DATA). 
After the "post" process, you must safely backup the 
generated label file and (usually) the WAL files created during the backup.

This version writes to `$PGDATA/base/pgsql_tmp`. If that directory is 
unavailable or read-only, the pre-step will fail immediately.
If there are any configuration variables set to unsafe values, 
the init function will fail. 

"Fail" does not mean it will exit, but will return with an error.

The functions override the trap EXIT, but tries to preserve anything
you may have already registered there.

WARNING: multiple tablespaces are not yet supported.

## Requirements ## 

 * Postgresql >= 10
 * Bash >= v4.0
 
## Usage ##

Edit `config.sh` and adapt the variables to your needs. Rename it, move it as you see 
fit. 

Next, in your backup script, scource `pg_filebased_backup.sh`.  

In your script, you must invoke the following functions:

   pg_fbb_init <config-file-path>

   pg_fbb_pre [<backup-label>]
   
   pg_fbb_post [<backup-label>]

If "pre" and "post" are run as separate scripts, invoke pg_fbb_init in both, 
then call the "pre" or "post" as relevant. 

When creating a monolithic backup script, you should do the all of the following:

1. Source the script.
2. Invoke `pg_fbb_init` as above.
3. Do other script initialization
4. Invoke `pg_fbb_pre`
5. Do other backup steps (eg, create volume snapshot)
6. Invoke `pg_fbb_post`
7. Backup the file specified by the output of `pg_fbb_get_label_file`.
8. Backup all files contained in the WAL_list file, which 
   is located at the output of `pg_fbb_get_wal_list`. 
   If you are already performing wal-log archiving, you can skip
   this step.
9. Remove the label file and wal-list file. 

## Implementation details

The "pre" step starts a coprocess which will run in the background. 
That coprocess will in turn spawn its own coprocess, specifically
for communicating with and keeping an open connection to postgres.
If that connection is terminated, the non-exclusive backup mode
will fail.
  
The "pre" step will write out
a backup label in `$PGDATA/base/pgsql_tmp` suffixed with the label 
you provided (or defaulted to). When "post" is called, and no label is
provided, it will look in that directory for the most recently created 
label and use it.  Thus, if you run multiple backups concurrently, 
you must specify the label, which should obviously be unique.  Sometimes,
the $PPID (parent ID) of the script should be enough.

The "pre" stage performs the following:

   1. Execute postgresql's internal start-backup function,
      ie, `pg_start_backup()`. 
   2. Wait until the backup-label file in `$PGDATA/base/pgsql_tmp`
      to exist, or USR1 signal received. (Caution -- all other 
      signals will force an exit of the process.)
      A sleep 1 loop is used for the wait. This time can 
      be changed using `$PG_FBB_POLL_TIME`.
   3. Calls `pg_stop_backup()` and saves the relevant 
      output to the backup-label file. 
   4. closes the postgres connection and exits itself.

The "post" stage performs the following steps:

   1. Signify to the co-process that it should tell postgresql to invoke
      the `pg_stop_backup()`. If in the same process, USR1 is sent to 
      it, otherwise, the backup-label file is created. 

   2. Wait for the backup-label file to become non-zero in size.
      A sleep loop is used, in the same way the pre stage does it.

   3. Attempts to determine the list of WALs changed since
      the backup began and then creates the WAL_LIST file containing
      the names (note, not the paths, just the names).
      The file is saved in `$PG_FBB_WAL_LIST` and defaults
      to `$PGDATA/base/pgsql_tmp/wal_list.<label>`.
      The current implementation uses some variation of 
      `find ... -newerct @<start-time> \! -newerct @<end-time>`
      You might end up backing one more file than needed.
  
   4. Clears the trap handler. 

If the EXIT trap is triggered (via other trap signal or via `exit`),
it will kill the postgresql corprocess scripts, if any remain. If 

## Note
  
If you are using tablespaces, you'll have to collect the contents of the
`tablespace_map` file from the column of that name in the `backup` table
and also store it with the backup.

## Configuration ##

The scripts are configured by editing `config.sh`.

The environment variables `PGHOST`, `PGPORT`, `PGDATABASE` and `PGUSER`
determine which database cluster is backed up. (Why DATABASE?)

* PG_FBB_WAL_LIST     -- you can use `%s` where the label will go.
* PG_FBB_POLL_TIME    -- argument to `sleep` in the wait-loop, default 1 (s)
* PG_FBB_TIMEOUT      -- The `pg_start_backup` can take a long time. Set this
                         to make sure it does not wait forever. Default: 23h

  
