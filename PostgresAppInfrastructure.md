[Technical Approach Page](PostgresTechnicalApproach)

This page describes the work needed to facilitate using both Oracle and `PostgreSQL` within the Spacewalk application.
### Installation



 * Driver packaging
 * spacewalk-setup
 * db-control
### Schema Management



The _common_ schema approach is the goal for managing the table/index DDL files.  The following describes how this can be accomplished 
with a templating or _schema generator_ tool that would be used at __build__ time to create the DB specific .sql files from the _common' schema files.  The team has _hand'' created the postgres .sql files
as both a fall back and an example of what the end schema needs to be for postgres.  Should the _common_ schema approach prove to be too complex or unmanageable in
some way, we can always manage duplicate schema files.

 * /schema/spacewalk
    Contains DDL files.
 * */schema/spacewalk/common*
    Common schema.
 * /schema/spacewalk/common/tables
    Although we have created ''static'' postgres table DDL (.sql) files, the long term goal is to have most of the table .sql files would go in this directory.  The ''common'' schema syntax would be a superset of both DDL grammars.  At build time, the ''common'' schema would be used to generate the ''dynamic'' DDL files for each DB in the DB specific directories.  Also contains the ''common'' tables.deps file.
 * /schema/spacewalk/common/views
    Although we have created ''static'' postgres view DDL (.sql) files, the long term goal is to have most of the view .sql files would go in this directory.  The ''common'' schema syntax would be a superset of both DDL grammars.  At build time, the ''common'' schema would be used to generate the ''dynamic'' DDL files for each DB in the DB specific directories.  In order for a view to be common, it must contain a query that works for all databases.  Also contains the ''common'' views.deps file.
 * /schema/spacewalk/common/data
    Although we have created ''static'' postgres data loading (.sql) files (inserts), the long term goal is to have most of the data loading .sql files would go in this directory.  The ''common'' schema syntax would be a superset of both DML grammars.  At build time, the ''common'' schema would be used to generate the ''dynamic'' DDL files for each DB in the DB specific directories.  Also contains the ''common'' data.deps file.
 * */schema/spacewalk/oracle*
    Contains both: generated (dynamic) '''oracle''' specific schema and ''forked'' (static) schema files.
 * /schema/spacewalk/oracle/class
    Contains '''oracle''' specific user defined types  (such as EVR_T) DDL files.
 * /schema/spacewalk/oracle/types
    Contains '''oracle''' specific user defined types DDL files.
 * /schema/spacewalk/oracle/tables
    Contains '''oracle''' specific ''(forked)'' table DDL files.
 * /schema/spacewalk/oracle/tables/common
    Contains '''common'''  table DDL and (.dep) files.  Populated at build.
 * /schema/spacewalk/oracle/views
    Contains '''oracle''' specific ''(forked)'' view DDL files.
 * /schema/spacewalk/oracle/views/common
    Contains '''common'''  view DDL and (.dep) files.  Populated at build.
 * /schema/spacewalk/oracle/data
    Contains '''oracle''' specific ''(forked)'' data loading (insert) files.
 * /schema/spacewalk/oracle/data/common
    Contains '''common''' data loading (insert) and (.dep) files.  Populated at build.
 * /schema/spacewalk/oracle/triggers
    Contains '''oracle''' specific ''forked'' trigger creation files.
 * /schema/spacewalk/oracle/procs
    Contains '''oracle''' specific ''forked'' stored procedure creation files.
 * /schema/spacewalk/oracle/packages
    Contains '''oracle''' specific ''forked'' package/package body creation files.
 * /schema/spacewalk/oracle/synonyms
    Contains '''oracle''' specific ''forked'' synonym creation files (although the plan is to get rid of these).
 * */schema/spacewalk/postgres*
    Contains both: generated (dynamic) '''postgres''' specific schema and ''forked'' (static) schema files.
 * /schema/spacewalk/postgres/class
    Contains '''postgres''' specific user defined types  (such as EVR_T) DDL files.
 * /schema/spacewalk/postgres/types
    Contains '''postgres''' specific user defined types DDL files.
 * /schema/spacewalk/postgres/tables
    Contains '''postgres''' specific ''(forked)'' table DDL files.
 * /schema/spacewalk/postgres/tables/common
    Contains '''common'''  table DDL and (.dep) files.  Populated at build.
 * /schema/spacewalk/postgres/views
    Contains '''postgres''' specific ''(forked)'' view DDL files.
 * /schema/spacewalk/postgres/views/common
    Contains '''common'''  view DDL and (.dep) files.  Populated at build.
 * /schema/spacewalk/postgres/data
    Contains '''postgres''' specific ''(forked)'' data loading (insert) files.
 * /schema/spacewalk/postgres/data/common
    Contains '''common''' ''(generated)'' data loading (insert) and (.dep) files.  Populated at build.
 * /schema/spacewalk/postgres/triggers
    Contains '''postgres''' specific ''forked'' trigger creation files.
 * /schema/spacewalk/postgres/procs
    Contains '''postgres''' specific ''forked'' stored procedure creation files.
 * /schema/spacewalk/postgres/packages
    Contains '''postgres''' specific ''forked'' package/package body creation files.
### Database Upgrade



Example directories for upgrade listed to demonstrate concepts.  Having common upgrade DDL that is used to generate
the DB specific upgrade DDL (like the table schema) may not be feasible but a worthy goal.  Any de duplication will be good.

*note:* We need some work here.

 * /schema/spacewalk/upgrade/spacewalk_04-05
 * /schema/spacewalk/upgrade/spacewalk_04-05/common
 * /schema/spacewalk/upgrade/spacewalk_04-05/oracle
 * /schema/spacewalk/upgrade/spacewalk_04-05/postgres
### Java Stack



 * Driver setup
   * The _/etc/rhn/rhn.conf_ file needs to have the proper driver specified (see spacewalk-setup).
 * Handling _forked_ queries.
   * Queries are already resolved by name in the Java stack.  So, the work to be done here is to provide for _forked_ queries using a namespace prefix.  Common queries would retain the _plain_ name as the do today.  _Forked_ queries would have their name qualified by a prefixed matching the database _type_ property defined in the _/etc/rhn/rhn.conf_ file.  An oracle specific query named "listAllChannels" when forked would be qualified as: "oracle:listAllChannels" and "postgres:listAllChannels".  This qualification would be transparent to the caller.  The underlying query resolution mechanism would first try to resolve as a common query then try the DB specific prefix.  The hunting order can be reversed for performance if it turns out there are more forked queries then common (although this would be contrary to our goal).
     * Hibernate queries
     * Datasource queries
### Python Stack



 * Driver setup (DONE, rhnSQL checks for db backend to instantiate in rhn.conf)
 * Handling _forked_ queries.
   * Queries are __not__ currently resolved by name in the Python stack.  So, the work to be done here is to add this capability into rhnSQL and provide for _forked_ queries using a namespace prefix.  An approach could be to add a python _package_ containing one or more python _modules_ containing a dictionary of named queries.  _Forked_ queries would have their name qualified by a prefixed matching the database _type_ property defined in the _/etc/rhn/rhn.conf_ file.  An oracle specific query named "listAllChannels" when forked would be qualified as: "oracle:listAllChannels" and "postgres:listAllChannels".  This qualification would be transparent to the caller.  The underlying query resolution mechanism would first try to resolve as a common query then try the DB specific prefix.  The hunting order can be reversed for performance if it turns out there are more forked queries then common (although this would be contrary to our goal).  Unlike the Java stack, __only__ the _forked_ queries would be added to the dictionaries and the calling code modified to perform the lookup.
### Perl Stack



 * Driver setup
   * Perl DB code must be modified to know to instantiate Oracle or PostgreSQL driver using settings in rhn.conf.
 * Handling _forked_ queries.
   * Queries are already __mostly__ resolved by name in the Perl stack.  So, the work to be done here is to provide for _forked_ queries using a namespace prefix.  Common queries would retain the _plain_ name as the do today.  _Forked_ queries would have their name qualified by a prefixed matching the database _type_ property defined in the _/etc/rhn/rhn.conf_ file.  An oracle specific query named "listAllChannels" when forked would be qualified as: "oracle:listAllChannels" and "postgres:listAllChannels".  This qualification would be transparent to the caller.  The underlying query resolution mechanism would first try to resolve as a common query then try the DB specific prefix.  The hunting order can be reversed for performance if it turns out there are more forked queries then common (although this would be contrary to our goal).  Queries embedded in the code will have to be moved to using the lookup mechanism.
### Tasks



||*ID*|| *Description* || *Component* || *Status* || *Assigned* || *Notes* ||
|| I1 || Modify spacewalk-setup || installer || started || dgoodwin || ||||
|| I2 || Create _postgres_ version of db-control || admin || || ||||
|| I3.1 || Schema generator || schema || prototyped || jortel || most grammar supported ||
|| I3.2 || Change postgres NUMERIC to BIGINT || schema || || jortel || ||||
|| I3.3 || Should resize NUMBER(38) to something reasonable ( like NUMBER(9) ) in oracle || schema || || jortel || need to run by community || nice to have ||
|| I4 || Add support for forked queries: hibernate || java || || jortel || ||||
|| I5 || Add support for forked queries: datasource || java || || jortel || ||||
|| I6 || Add support for named queries || python || ||  || ||||
|| I7 || Move embedded SQL to named queries || python || ||  || ||||
|| I8 || Add support for forked queries: datasource || perl || || shughes :) || ||||
|| I9 || Move embedded SQL to datasource || perl || || shughes :) || ||||
|| I10 || RPM Packaging || rpm || || || ||||