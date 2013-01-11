PostgreSQL Partition Manager Extension (pg_partman)
===================================================

About
-----
Extension to help make managing time or serial id based table partitioning easier.

For this extension, most of the attributes of the child partitions are all obtained from the original parent. This includes defaults, indexes (primary keys, unique, etc) & constraints. Permissions are not yet handled, but are coming soon. While you would not normally create indexes on the parent of a partition set, doing so makes it much easier to manage in this case. There will be no data in the parent table (if everything is working right), so they will not take up any space or have any impact on system performance. Using the parent table as a control to the details of the child tables gives a more central place to manage things that's a little more natural than a configuration table or using setup functions.

If you don't need to keep data in older partitions, a retention system is available to automatically drop unneeded child partitions. By default, they are only uninherited not actually dropped, but that can be configured if desired. If the old partitions are kept, dropping their indexes can also be configured to recover disk space. Note that this will also remove any primary key or unique constraints in order to allow the indexes to be dropped. To set the retention policy, enter either an interval or integer value into the **retention** column of the **part_config** table. For time-based partitioning, the interval value will set that any partitions containing data older than that will be dropped. For id-based partitioning, the integer value will set that any partitions with an id value less than the current maximum id value minus the retention value will be dropped. For example, if the current max id is 100 and the retention value is 30, any partitions with id values less than 70 will be dropped. The current maximum id value at the time the drop function is run is always used.

The PG Jobmon extension (https://github.com/omniti-labs/pg_jobmon) is optional and allows auditing and monitoring of partition maintenance. If jobmon is installed and configured properly, it will automatically be used by partman with no additional setup needed. By default, any function that fails to run successfully 3 consecutive times will cause jobmon to raise an alert. This is why the default pre-make value is set to 4 so that an alert will be raised in time for intervention with no additional configuration of jobmon needed. You can of course configure jobmon to alert before (or later) than 3 failures if needed. If you're running partman in a production environment it is HIGHLY recommended to have jobmon installed and some sort of 3rd-party monitoring configured with it to alert when partitioning fails (Nagios, Circonus, etc).

Functions
---------
All functions are run with SECURITY DEFINER, so just ensure that the function owner has the needed privileges for table & function creation and removal.

*create_parent(p_parent_table text, p_control text, p_type part.partition_type, p_interval text, p_premake int DEFAULT 3, p_debug boolean DEFAULT false)*
 * Main function to create a partition set with one parent table and inherited children. Parent table must already exist. Please apply all indexes & constraints to parent table so they will propagate to children (permissions not yet propagating; working on it!).
 * An ACCESS EXCLUSIVE lock is taken on the parent table during the running of this function. No data is moved when running this function, so lock should be brief.
 * p_parent_table - the existing parent table
 * p_control - the column that the partitioning will be based on. Must be a time or integer based column.
 * p_type - one of 4 values to set the partitioning type that will be used
 
 > **time-static** - Trigger function inserts only into specifically named partitions (handles data for current partition, 2 partitions ahead and 1 behind).  Cannot handle inserts to parent table outside the hard-coded time window. Function is kept up to date by run_maintenance() function. Ideal for high TPS tables that get inserts of new data only.  
 > **time-dynamic** - Trigger function can insert into any child partition based on the value of the control column. More flexible but not as efficient as time-static. Be aware that if the appropriate partition doesn't yet exist for the data inserted, the insert will fail.  
 > **id-static** - Same functionality as time-static but for a numeric range instead of time. When the id value has reached 50% of the max value for that partition, it will automatically create the next partition in sequence if it doesn't yet exist. Does NOT require run_maintenance() function to create new partitions.  
 > **id-dynamic** - Same functionality and limitations as time-dynamic but for a numeric range instead of time. Uses same 50% rule as id-static to create future partitions. Does NOT require run_maintenance() function to create new partitions.  

 * p_interval - the time or numeric range interval for each partition. Supported values are:

 > **yearly** - One partition per year  
 > **quarterly** - One partition per yearly quarter. Partitions are named as YYYYqQ (ex: 2012q4)  
 > **monthly** - One partition per month  
 > **weekly** - One partition per week. Follows ISO week date format (http://en.wikipedia.org/wiki/ISO_week_date). Partitions are named as IYYYwIW (ex: 2012w36)  
 > **daily** - One partition per day  
 > **hourly** - One partition per hour  
 > **half-hour** - One partition per 30 minute interval on the half-hour (1200, 1230)  
 > **quarter-hour** - One partition per 15 minute interval on the quarter-hour (1200, 1215, 1230, 1245)  
 > **<integer>** - For ID based partitions, the integer value range of the ID that should be set per partition. This is the actual integer value, not text values like time-based partitioning.  

 * p_premake - is how many additional partitions to always stay ahead of the current partition. Default value is 4. This will keep at minimum 5 partitions made, including the current one. For example, if today was Sept 6, 2012, and premake was set to 4 for a daily partition, then partitions would be made for the 6th as well as the 7th, 8th, 9th and 10th. 
 * p_debug - turns on additional debugging information (not yet working).

*run_maintenance()*
 * Run this function as a scheduled job (cronjob, etc) to automatically keep time-based partitioning working.
 * Run this function as a scheduled job to automatically drop child tables and/or indexes if the retention options are configured.
 * Every run checks all tables listed in the **part_config** table with the types **time-static** and **time-dynamic** to pre-make child tables (not needed for id-based partitioning).
 * Every run checks all tables of all types listed in the **part_config** table with a value in the **retention** column and drops tables as needed (see **About** and config table below).
 * Will automatically pre-create the new partition, keeping ahead by the number set as the **premake**. 
 * Will automatically update the function for **time-static** partitioning to keep the parent table pointing at the correct partitions. When using time-static, run this function more often than the partitioning interval to keep the function running its most efficient. For example, if using quarter-hour, run every 5 minutes; if using daily, run at least twice a day, etc.

*create_prev_time_partition(p_parent_table text, p_batch int DEFAULT 1) RETURNS bigint*
 * This function is used to partition data that may have existed prior to setting up the parent table as a time-based partition set. 
 * If you are partitioning a large amount of previous data, it's recommended to run this function with an external script with small batch amounts. This will prevent a failure from causing an extensive rollback and avoid transactional locks.
 * p_parent_table - the existing parent table. This is assumed to be where the unpartitioned data is located.
 * p_batch - how many partitions will be made in a single run of the function. Default value is 1.
 * Returns the number of rows that were moved from the parent table to partitions. Returns zero when parent table is empty and partitioning is complete.

*create_prev_id_partition(p_parent_table text, p_batch int DEFAULT 1) RETURNS bigint*
 * This function is used to partition data that may have existed prior to setting up the parent table as a serial id partition set. 
 * If you are partitioning a large amount of previous data, it's recommended to run this function with an external script with small batch amounts. This will prevent a failure from causing an extensive rollback and avoid transactional locks.
 * p_parent_table - the existing parent table. This is assumed to be where the unpartitioned data is located.
 * p_batch - how many partitions will be made in a single run of the function. Default value is 1.
 * Returns the number of rows that were moved from the parent table to partitions. Returns zero when parent table is empty and partitioning is complete.

*check_parent() RETURNS SETOF (parent table, count)*
 * Run this function to monitor that the parent tables of the partition sets that pg_partmon manages do not get rows inserted to them.
 * Returns a row for each parent table along with the number of rows it contains. Returns zero rows if none found.

*drop_time_partition(p_parent_table text, p_keep_table boolean DEFAULT NULL, p_keep_index boolean DEFAULT NULL)*
 * This function is used to drop child tables from a time-based partition set. The table is by default just uninherited and not actually dropped. It is recommended to use the **run_maintenance()** function to manage dropping old child tables. Function can be called directly if needed.
 * p_parent_table - the existing parent table of a time-based partition set.
 * p_keep_table - optional parameter to tell partman whether to keep or drop the table in addition to uninheriting it. TRUE means the table will not actually be dropped; FALSE means the table will be dropped. This function will just use the value configured in **part_config** if not explicitly set.
 * p_keep_index - optional parameter to tell partman whether to keep or drop the indexes of the child table when it is uninherited. TRUE means the indexes will be kept; FALSE means all indexes will be dropped. This function will just use the value configured in **part_config** if not explicitly set. This option is ignored if p_keep_table is set to FALSE.

*drop_id_partition(p_parent_table text, p_keep_table boolean DEFAULT NULL, p_keep_index boolean DEFAULT NULL)*
 * This function is used to drop child tables from an id-based partition set. The table is by default just uninherited and not actually dropped. It is recommended to use the **run_maintenance()** function to manage dropping old child tables. Function can be called directly if needed.
 * p_parent_table - the existing parent table of a time-based partition set.
 * p_keep_table - optional parameter to tell partman whether to keep or drop the table in addition to uninheriting it. TRUE means the table will not actually be dropped; FALSE means the table will be dropped. This function will just use the value configured in **part_config** if not explicitly set.
 * p_keep_index - optional parameter to tell partman whether to keep or drop the indexes of the child table when it is uninherited. TRUE means the indexes will be kept; FALSE means all indexes will be dropped. This function will just use the value configured in **part_config** if not explicitly set. This option is ignored if p_keep_table is set to FALSE.


Tables
------
*part_config*  
    Stores all configuration data for partition sets mananged by the extension. The only columns in this table that should ever need to be manually changed are  
    **retention**, **retention_keep_table**, & **retention_keep_index** to set the partition set's retention policy.   
    The rest are managed by the extension itself and should not be changed unless absolutely necessary.

    parent_table            - Parent table of the partition set
    type                    - Type of partitioning. An enum of the types listed above in create_parent() function section.
    part_interval           - Text type value that determines the interval for each partition. 
                              Must be a value that can either be cast to the interval or bigint data types.
    control                 - Column used as the control for partition constraints. Must be a time or integer based column.
    premake                 - How many partitions to keep pre-made ahead of the current partition. Default is 4.
    retention               - Text type value that determines how old the data in a child partition can be before it is dropped. 
                              Must be a value that can either be cast to the interval or bigint data types. 
                              Leave this column NULL (the default) to always keep all child partitions. See **About** section for more info.
    retention_keep_table    - Boolean value to determine whether dropped child tables are kept or actually dropped. 
                              Default is TRUE to keep the table and only uninherit it. Set to FALSE to have the child tables removed from the database completely.
    retention_keep_index    - Boolean value to determine whether indexes are dropped for child tables that are uninherited. 
                              Default is TRUE. Set to FALSE to have the child table's indexes dropped when it is uninherited.
    datetime_string text    - For time-based partitioning, this is the datetime format string used when naming child partitions. 
                              **DO NOT CHANGE THIS VALUE** or you will break automated partition management.
    last_partition text     - The last successfully created partition.
                              **DO NOT CHANGE THIS VALUE** or you will break automated partition management.
