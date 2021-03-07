# SQLdb360

SQLdb360 is a "free to use toolset" to perform an initial assessment of an entire Oracle database or a particular SQL statement.
SQLdb360 is made of two independent tools, eDB360 (database-wide analysis) and SQLd360 (individual SQL analysis).


(README edits by Karl Arao)

# Execute SQLD360 in batch mode

## 1) Download the tool 
https://github.com/karlarao/sqldb360/archive/master.zip
```
$ wget https://github.com/karlarao/sqldb360/archive/master.zip
$ unzip master.zip
$ cd sqldb360-master/
```

## 2) Run the tool in batch mode 
```
-- on SQL prompt copy pase the following

-- login as SYSDBA or any privileged user (DBA role)
SQL> sqlplus "/ as sysba" 

-- You can do a batch run of report generation by inserting the SQL_ID to the plan_table. 
-- You can do this by doing manual INSERT or INSERT..SELECT on a filtered list of SQL_IDs. 
-- Examples below: 


------------------------------------
-- Manual INSERT 
------------------------------------

-- required input variables
def edb360_secs2go = 3600
def psqlid = "sqld360batch"
-- insert
INSERT INTO plan_table (id, statement_id, operation, options, object_node) values (dbms_random.value(1,10000), 'SQLD360_SQLID', '0p1f7w41jj1tq', '111007', NULL);
INSERT INTO plan_table (id, statement_id, operation, options, object_node) values (dbms_random.value(1,10000), 'SQLD360_SQLID', '3ddvj44c10qqx', '111007', NULL);
INSERT INTO plan_table (id, statement_id, operation, options, object_node) values (dbms_random.value(1,10000), 'SQLD360_SQLID', '3m8smr0v7v1m6', '111007', NULL);
INSERT INTO plan_table (id, statement_id, operation, options, object_node) values (dbms_random.value(1,10000), 'SQLD360_SQLID', '3nnj1js6gy2yb', '111007', NULL);
INSERT INTO plan_table (id, statement_id, operation, options, object_node) values (dbms_random.value(1,10000), 'SQLD360_SQLID', '3sqgkcng6vx8r', '111007', NULL);
-- run SQLD360
@sqld360.sql 


------------------------------------
-- or run the INSERT..SELECT below
------------------------------------

-- required input variables
def edb360_secs2go = 3600
def psqlid = "sqld360batch"
-- insert..select
INSERT INTO plan_table (id, statement_id, operation, options, object_node) 
select dbms_random.value(1,10000), 'SQLD360_SQLID', sql_id, '111007', NULL from (
select sql_id,sql_type,module,parsing_schema,
       count(sql_plan_hash_value) distinct_phv,
                count(*) exec_count,
        round(avg(EXTRACT(HOUR FROM run_time) * 3600
                    + EXTRACT(MINUTE FROM run_time) * 60
                    + EXTRACT(SECOND FROM run_time)),2) elap_avg ,
        round(min(EXTRACT(HOUR FROM run_time) * 3600
                    + EXTRACT(MINUTE FROM run_time) * 60
                    + EXTRACT(SECOND FROM run_time)),2) elap_min ,
        round(max(EXTRACT(HOUR FROM run_time) * 3600
                    + EXTRACT(MINUTE FROM run_time) * 60
                    + EXTRACT(SECOND FROM run_time)),2) elap_max,
                    round(avg(pct_cpu),0) pct_cpu,
                    round(avg(pct_wait),0) pct_wait,
                    round(avg(pct_io),0) pct_io,                    
                    round(max(temp)/1024/1024,2) max_temp_mb,
                    round(max(pga)/1024/1024,2) max_pga_mb,
                    round(max(rbytes)/1024/1024,2) max_read_mb,
                    round(max(wbytes)/1024/1024,2) max_write_mb,
                    max(riops) max_riops,
                    max(wiops) max_wiops
from  (
        select
                       ash.sql_id sql_id,
                       aud.name sql_type, 
                       ash.module module,
                       pschema.username parsing_schema,
                       ash.sql_exec_id sql_exec_id,
                       ash.sql_plan_hash_value sql_plan_hash_value,
                       max(ash.sql_exec_start) sql_exec_start,
                       max(ash.sample_time - ash.sql_exec_start) run_time,
                       max(ash.TEMP_SPACE_ALLOCATED) temp,
                       max(ash.PGA_ALLOCATED) pga,
                       max(ash.DELTA_READ_IO_BYTES) rbytes,
                       max(ash.DELTA_READ_IO_REQUESTS) riops,
                       max(ash.DELTA_WRITE_IO_BYTES) wbytes,
                       max(ash.DELTA_WRITE_IO_REQUESTS) wiops,
                       round(sum(decode(ash.session_state,'ON CPU',1,0))/sum(decode(ash.session_state,'ON CPU',1,1)),2)*100 pct_cpu,
                       round((sum(decode(ash.session_state,'WAITING',1,0)) - sum(decode(ash.session_state,'WAITING', decode(ash.wait_class, 'User I/O',1,0),0)))/sum(decode(ash.session_state,'ON CPU',1,1)),2)*100 pct_wait,
                       round((sum(decode(ash.session_state,'WAITING', decode(ash.wait_class, 'User I/O',1,0),0)))/sum(decode(ash.session_state,'ON CPU',1,1)),2)*100 pct_io
        from
               dba_hist_active_sess_history ash, audit_actions aud, all_users pschema
        where
               ash.sql_exec_start is not null
               and ash.sql_opcode=aud.action
               and ash.user_id = pschema.user_id 
--               and sql_id = '0nx0wbcfa71gf'
               and pschema.username in ('SCOTT','HR','APP_SCHEMA1','APP_SCHEMA2')
--               and pschema.username not in ('SYS','SYSTEM','DBSNMP','SYSMAN','AUDSYS','MDSYS','ORDSYS','XDB','APEX_PUBLIC_USER','ORACLE_OCM','APEX_050100','GSMADMIN_INTERNAL','ORDS_METADATA','XFILES','MYDBA','XDBEXT')
        group by ash.sql_id,aud.name,ash.module,pschema.username,ash.sql_exec_id,ash.sql_plan_hash_value
       )
--where pct_cpu > 90
--where pct_io > 90
--where pct_wait > 90
group by sql_id,sql_type,module,parsing_schema
order by elap_max desc nulls last
--order by max_read_mb desc nulls last
)
where rownum < 6;
-- run SQLD360
@sqld360 
```


## SQLD360 example output 
<img width="1166" src="https://user-images.githubusercontent.com/3683046/110233253-d83c1880-7ef0-11eb-8478-f6968adf4c4a.png">


# EDB360 

```
edb360 sections 

1a. Database Configuration
1b. Security
1c. Auditing
1d. Memory
1e. Resources (as per AWR and MEM)
1f. Resources (as per Statspack)

2a. Database Administration
2b. Storage
2c. Automatic Storage Management (ASM)
2d. Backup and Recovery

3a. Database Resource Management (DBRM)
3b. Plan Stability
3c. Cost-based Optimizer (CBO) Statistics
3d. Performance Summaries
3e. Operating System (OS) Statistics History
3h. Sessions
3i. JDBC Sessions
3j. Non-JDBC Sessions
3k. Data Guard Primary Site

4a. System Global Area (SGA) Statistics History
4b. Program Global Area (PGA) Statistics History
4c. Memory Statistics History
4d. System Time Model
4e. System Time Model Components
4f. Wait Times and Latency per Class
4g. Latency Histogram for Top 24 Wait Events
4h. Average Latency for Top 24 Wait Events
4i. Waits Count v.s. Average Latency for Top 24 Wait Events
4j. Parallel Execution
4k. System Metric History per Snap Interval
4l. System Metric Summary per Snap Interval

5a. Active Session History (ASH)
5b. Active Session History (ASH) on Wait Class
5c. Active Session History (ASH) on CPU and Top 24 Wait Events
5d. System Statistics per Snap Interval
5e. System Statistics (Exadata) per Snap Interval
5f. System Statistics (Current) per Snap Interval
5g. Exadata

6a. Active Session History (ASH) - Top Timed Classes
6b. Active Session History (ASH) - Top Timed Events
6c. Active Session History (ASH) - Top SQL
6d. Active Session History (ASH) - Top SQL - Time Series
6e. Active Session History (ASH) - Top Programs
6f. Active Session History (ASH) - Top Modules and Actions
6g. Active Session History (ASH) - Top Users
6h. Active Session History (ASH) - Top PLSQL Procedures
6i. Active Session History (ASH) - Top Data Objects
6j. Active Session History (ASH) - Service and User
6k. Active Session History (ASH) - Top PHV
6l. Active Session History (ASH) - Top Signature

7a. AWR/ADDM/ASH Reports
7b. SQL Sample

****************************************************************************************

Example runs:

Execute everything 
@edb360.sql T NULL

Execute only sections 1 to 6
@edb360.sql T 1-6

Execute only sections 1 to 7a
@edb360.sql T 1-7a

Execute only section 7
@edb360.sql T 7
```







(Original README below)

## Download
Remember to download the lastest *stable* release under "Releases". If you just use the download link you'll get the unstable version, newest features and fixes, but unstable

## Steps

1. Download the tool into target database server
2. Navigate to master directory and connect into SQLPlus as DBA or user with access to data dictionary
3. Execute edb360.sql (database view) or sqld360.sql (one SQL focus)
   - Both tools will prompt for the license available on the target database.
     - [T | D | N] For Tuning, Diagnostics or None
   - Both tools accept an optional configuration file
   - SQLd360 requires the SQL ID of interest to be provided
4. Copy output zip to client (PC or Mac), unzip and open in browser file 00001_*_index.html

## Notes

1. eDB360 and SQLd360 run transparently on and support RAC, Exadata and In-Memory. In a multitenant environment, connect to PDB of interest.
2. No application data is collected, only metadata is accessed.
3. Both tools work in a "no evidence left behind" fashion, meaning there is no post execution step that needs to be executed, the tools clean after themselves.
4. It is recommended to download the latest version of the tool before using it, this is to minimize the impact of known bugs and benefit from latest features.

## Troubleshooting

edb360 takes up to 24 hours to execute on a large database. On smaller ones or on Exadata it may take a few hours or less. In rare cases it may require even more than 24 hrs.
By default, eDB360 executes a pre-check and asks for confirmation in case the execution is estimated to take more than 8 hours.
Multiple options are available to speed up large executions, for details refer to https://carlos-sierra.net/2017/04/10/edb360-meets-eadam-3-0-when-two-heads-are-better-than-one/

SQLd360 is generally faster, given the reduced scope, and as such no pre-check is executed.

## Contacts

For questions, feedbacks or issues please contact sqldb360@gmail.com

## License

  SQLdb360 - An open-source tool to diagnose Oracle Databases and SQL
  statements - originally developed by Carlos Sierra and Mauro Pagano.

  This program is free software: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by
  the Free Software Foundation, either version 3 of the License, or
  (at your option) any later version.

  This program is distributed in the hope that it will be useful,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  GNU General Public License for more details.

  You should have received a copy of the GNU General Public License
  along with this program.  If not, see <http://www.gnu.org/licenses/>.
