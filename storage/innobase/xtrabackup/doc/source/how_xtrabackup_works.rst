================================
 How |Percona XtraBackup| Works
================================

|Percona XtraBackup| is based on :term:`InnoDB`'s crash-recovery functionality. It copies your |InnoDB| data files, which results in data that is internally inconsistent; but then it performs crash recovery on the files to make them a consistent, usable database again.

This works because |InnoDB| maintains a redo log, also called the transaction log. This contains a record of every change to InnoDB's data. When |InnoDB| starts, it inspects the data files and the transaction log, and performs two steps. It applies committed transaction log entries to the data files, and it performs an undo operation on any transactions that modified data but did not commit.

|Percona XtraBackup| works by remembering the log sequence number (:term:`LSN`) when it starts, and then copying away the data files. It takes some time to do this, so if the files are changing, then they reflect the state of the database at different points in time. At the same time, |Percona XtraBackup| runs a background process that watches the transaction log files, and copies changes from it. |Percona XtraBackup| needs to do this continually because the transaction logs are written in a round-robin fashion, and can be reused after a while. |Percona XtraBackup| needs the transaction log records for every change to the data files since it began execution.

The above is the backup process. Next is the prepare process. During this step, |Percona XtraBackup| performs crash recovery against the copied data files, using the copied transaction log file. After this is done, the database is ready to restore and use.

The above process is implemented in the |xtrabackup| compiled binary program. The |innobackupex| program adds more convenience and functionality by also permitting you to back up |MyISAM| tables and :term:`.frm` files. It starts |xtrabackup|, waits until it finishes copying files, and then issues ``FLUSH TABLES WITH READ LOCK`` to prevent further changes to |MySQL|'s data and flush all |MyISAM| tables to disk. It holds this lock, copies the |MyISAM| files, and then releases the lock. 

.. note:: 

  |Percona XtraBackup| will use `Backup locks <https://www.percona.com/doc/percona-server/5.6/management/backup_locks.html#backup-locks>`_ where available as a lightweight alternative to ``FLUSH TABLES WITH READ LOCK``. This feature is available in |Percona Server| 5.6+. |Percona XtraBackup| uses this automatically to copy non-InnoDB data to avoid blocking DML queries that modify InnoDB tables. When backup locks are supported by the server, |xtrabackup| will first copy InnoDB data, run the ``LOCK TABLES FOR BACKUP`` and copy the |MyISAM| tables and :term:`.frm` files. After that |xtrabackup| will use ``LOCK BINLOG FOR BACKUP`` to block all operations that might change either binary log position or ``Exec_Master_Log_Pos`` or ``Exec_Gtid_Set`` (i.e. master binary log coordinates corresponding to the current SQL thread state on a replication slave) as reported by ``SHOW MASTER/SLAVE STATUS``. |xtrabackup| will then finish copying the REDO log files and fetch the binary log coordinates. After this is completed |xtrabackup| will unlock the binary log and tables. 

The backed-up |MyISAM| and |InnoDB| tables will eventually be consistent with each other, because after the prepare (recovery) process, |InnoDB|'s data is rolled forward to the point at which the backup completed, not rolled back to the point at which it started. This point in time matches where the ``FLUSH TABLES WITH READ LOCK`` was taken, so the |MyISAM| data and the prepared |InnoDB| data are in sync.

The |xtrabackup| and |innobackupex| tools both offer many features not mentioned in the preceding explanation. Each tool's functionality is explained in more detail on its manual page. In brief, though, the tools permit you to do operations such as streaming and incremental backups with various combinations of copying the data files, copying the log files, and applying the logs to the data.
