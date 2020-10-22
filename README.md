oracle active dataguard sop


-- 1. Create Physical standby DATABASE

-- Primary: all tables that aren’t supported for update
select * from DBA_LOGSTDBY_UNSUPPORTED_TABLE;

-- Primary: all tables that are not supported for update because of their datatypes
select * from DBA_LOGSTDBY_UNSUPPORTED;

-- to display a list of tables that SQL Apply may not be able to uniquely identify
select * from DBA_LOGSTDBY_NOT_UNIQUE;

-- 2. Stop Redo Apply on the Physical Standby Database
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

-- 3. Prepare the Primary Database to Support a Logical Standby Database
a. Set the LOG_ARCHIVE_DEST_n
alter system set log_archive_dest_4='LOCATION=/home/oracle/app/oracle/oradata/orcl/arch2 VALID_FOR=(STANDBY_LOGFILES, STANDBY_ROLE) db_unique_name=orcl' scope=both;
alter system set log_archive_dest_state_4='enable' scope=both;

alter system set log_archive_dest_5='LOCATION=/home/oracle/app/oracle/oradata/orcl/arch3 VALID_FOR=(STANDBY_LOGFILES, STANDBY_ROLE) db_unique_name=orcl' scope=both;
alter system set log_archive_dest_state_5='enable' scope=both;

b. Set the value of UNDO_RETENTION to 3600
alter system set undo_retention=3600 scope=both;

-- 4. Build a LogMiner Dictionary in the Redo Data
execute dbms_logstdby.build;

-- 5. Standby : Transition to a logical standby database
a. Execute the following command on the standby database to convert it to a logical standby database
alter database recover to logical standby <db_name>;

b. shut down the logical standby database instance and restart it in mount mode
shutdown immediate
startup mount

c. Adjust initialization parameters: LOG_ARCHIVE_DEST_n 
alter system set log_archive_dest_4='LOCATION=/home/oracle/app/oracle/oradata/orcl/arch2 VALID_FOR=(STANDBY_LOGFILES, STANDBY_ROLE) db_unique_name=preprod' scope=both;
alter system set log_archive_dest_state_4='enable' scope=both;

alter system set log_archive_dest_5='LOCATION=/home/oracle/app/oracle/oradata/orcl/arch3 VALID_FOR=(STANDBY_LOGFILES, STANDBY_ROLE) db_unique_name=orclstby' scope=both;
alter system set log_archive_dest_state_5='enable' scope=both;

-- 6. Open the Logical Standby DATABASE
a. Open the new logical standby database with the resetlogs option:
alter database open resetlogs;
b. Start the application of redo data to the logical standby database
alter database start logical standby apply immediate;

-- 7. Verify That the Logical Standby Database is Performing Properly
a. Verify that the archived redo log files were registered
select sequence#, first_time, next_time, dict_begin, dict_end
from dba_logstdby_log 
order by sequence#;
b. Begin sending redo data to the standby database
alter system archive log current;
c. query the DBA_LOGSTDBY_LOG view to verify that the archived redo log files were registeded
select * from dba_logstdby_log;
d. Verify that redo data is being applied correctly
select name, value from v$logstdby_stats
where name='coordinator state';
e. Query the v$logstdby_process view to see current sql apply activity
select sid,serial#,spid,type,high_scn
from v$logstdby_process;
f. Check the overall progress of sql apply
select applied_scn, latest_scn
from v$logstdby_progress;

-- 8. Startup Logical Stnadby Database After shutdown
startup;
alter database start logical standby apply immediate;