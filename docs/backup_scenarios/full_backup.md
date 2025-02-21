# The Backup Cycle - Full Backups

## Creating a backup

To create a backup, run **xtrabackup** with the `--backup`
option. You also need to specify the `--target-dir` option, which is where
the backup will be stored, if the *InnoDB* data or log files are not stored
in
the same directory, you might need to specify the location of those, too.
If the
target directory does not exist, *xtrabackup* creates it. If the directory
does
exist and is empty, *xtrabackup* will succeed.

*xtrabackup* will not overwrite existing files, it will fail with operating
system error 17, `file exists`.

To start the backup process run:

```
$ xtrabackup --backup --target-dir=/data/backups/
```

This will store the backup at `/data/backups/`. If you specify a
relative path, the target directory will be relative to the current
directory.

During the backup process, you should see a lot of output showing the data
files being copied, as well as the log file thread repeatedly scanning the
log
files and copying from it. Here is an example that shows the log thread
scanning the log in the background, and a file copying thread working on
the `ibdata1` file:

```
160906 10:19:17 Finished backing up non-InnoDB tables and files
160906 10:19:17 Executing FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS...
xtrabackup: The latest check point (for incremental): '62988944'
xtrabackup: Stopping log copying thread.
.160906 10:19:18 >> log scanned up to (137343534)
160906 10:19:18 Executing UNLOCK TABLES
160906 10:19:18 All tables unlocked
160906 10:19:18 Backup created in directory '/data/backups/'
160906 10:19:18 [00] Writing backup-my.cnf
160906 10:19:18 [00]        ...done
160906 10:19:18 [00] Writing xtrabackup_info
160906 10:19:18 [00]        ...done
xtrabackup: Transaction log of lsn (26970807) to (137343534) was copied.
160906 10:19:18 completed OK!
```

The last thing you should see is something like the following, where the
value
of the `<LSN>` will be a number that depends on your system:

```
$ xtrabackup: Transaction log of lsn (<SLN>) to (<LSN>) was copied.
```

!!! note
   
    Log copying thread checks the transactional log every second to see if there were any new log records written that need to be copied, but there is a chance that the log copying thread might not be able to keep up with the amount of writes that go to the transactional logs, and will hit an error when the log records are overwritten before they could be read.

After the backup is finished, the target directory will contain files such
as
the following, assuming you have a single InnoDB table `test.tbl1` and you
are using MySQL’s innodb_file_per_table option:

```
$ ls -lh /data/backups/
total 182M
drwx------  7 root root 4.0K Sep  6 10:19 .
drwxrwxrwt 11 root root 4.0K Sep  6 11:05 ..
-rw-r-----  1 root root  387 Sep  6 10:19 backup-my.cnf
-rw-r-----  1 root root  76M Sep  6 10:19 ibdata1
drwx------  2 root root 4.0K Sep  6 10:19 mysql
drwx------  2 root root 4.0K Sep  6 10:19 performance_schema
drwx------  2 root root 4.0K Sep  6 10:19 sbtest
drwx------  2 root root 4.0K Sep  6 10:19 test
drwx------  2 root root 4.0K Sep  6 10:19 world2
-rw-r-----  1 root root  116 Sep  6 10:19 xtrabackup_checkpoints
-rw-r-----  1 root root  433 Sep  6 10:19 xtrabackup_info
-rw-r-----  1 root root 106M Sep  6 10:19 xtrabackup_logfile
```

The backup can take a long time, depending on how large the database is. It
is
safe to cancel at any time, because *xtrabackup* does not modify the
database.

The next step is getting your backup ready to be restored.

## Preparing a backup

After making a backup with the `--backup` option, you need to
prepare it in order to restore it. Data files are not point-in-time
consistent
until they are *prepared*, because they were copied at different times as
the
program ran, and they might have been changed while this was happening.

If you try to start InnoDB with these data files, it will detect corruption
and
stop working to avoid running on damaged data. The `--prepare` step
makes the files perfectly consistent at a single instant in time, so you
can run
*InnoDB* on them.

You can run the *prepare* operation on any machine; it does not need to be
on the
originating server or the server to which you intend to restore. You can
copy
the backup to a utility server and prepare it there.

Note that *Percona XtraBackup* 8.0 can only prepare backups of *MySQL*
8.0, *Percona Server for MySQL* 8.0, and *Percona XtraDB Cluster* 8.0
databases. Releases prior to 8.0 are not supported.

During the *prepare* operation, *xtrabackup* boots up a kind of modified
embedded InnoDB (the libraries *xtrabackup* was linked against). The
modifications are necessary to disable InnoDB standard safety checks, such
as
complaining about the log file not being the right size. This warning is
not
appropriate for working with backups. These modifications are only for the
xtrabackup binary; you do not need a modified *InnoDB* to use *xtrabackup*
for
your backups.

The *prepare* step uses this “embedded InnoDB” to perform crash recovery on
the
copied data files, using the copied log file. The `prepare` step is very
simple to use: you simply run *xtrabackup* with the `--prepare` option
and tell it which directory to prepare, for example, to prepare the
previously
taken backup run:

```
$ xtrabackup --prepare --target-dir=/data/backups/
```

When this finishes, you should see an `InnoDB shutdown` with a message such
as the following, where again the value of LSN will depend on your
system:

```
InnoDB: Shutdown completed; log sequence number 137345046
160906 11:21:01 completed OK!
```

All following prepares will not change the already prepared data files,
you’ll
see that output says:

```
xtrabackup: This target seems to be already prepared.
xtrabackup: notice: xtrabackup_logfile was already used to '--prepare'.
```

It is not recommended to interrupt xtrabackup process while preparing
backup
because it may cause data files corruption and backup will become unusable.
Backup validity is not guaranteed if prepare process was interrupted.

!!! note
   
    If you intend the backup to be the basis for further incremental backups, you should use the `--apply-log-only` option when preparing the backup, or you will not be able to apply incremental backups to it. See the documentation on preparing incremental backups for more details.

## Restoring a Backup

!!! warning
   
    Backup needs to be prepared before it can be restored.

For convenience, *xtrabackup* binary has the `--copy-back`
option to copy the backup to the datadir of the server:

```
$ xtrabackup --copy-back --target-dir=/data/backups/
```

If you don’t want to save your backup, you can use the
`--move-back` option which will move the backed up data to
the datadir.

If you don’t want to use any of the above options, you can additionally use
**rsync** or **cp** to restore the files.

!!! note
   
    The datadir must be empty before restoring the backup. Also, it’s important to note that MySQL server needs to be shut down before restore is performed. You cannot restore to a datadir of a running mysqld instance (except when importing a partial backup).

Example of the **rsync** command that can be used to restore the backup
can look like this:

```
$ rsync -avrP /data/backup/ /var/lib/mysql/
```

You should check that the restored files have the correct ownership and
permissions.

As files’ attributes will be preserved, in most cases you will need to
change
the files’ ownership to `mysql` before starting the database server, as
they
will be owned by the user who created the backup:

```
$ chown -R mysql:mysql /var/lib/mysql
```

Data is now restored, and you can start the server.
