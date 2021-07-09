
## Extract Logs from Airflow DB for Datadog

The goal of the current approach is to get log data from the Airflow DB and write the records whenever there is a new insert in the log table. This method helps complement the Airflow metrics integration. 

Benefits:

* DB trigger sends the log to a file when a new DAG/Task event occurs
* The log is written as a JSON object, pipeline parsing is not needed for the most part
* The DD agent picks up the log
* Upon ingestion, the log is JSON formatted
* Creation of facets is easy
* Creating of dashboards is quick and straight forward. Use the JSON log info, in this article, for reference.
* The JSON Dashboard is included in this article

> Connect to psql & install the adminpack (if not yet installed)

```bash
root@3058510ae5f7:/# psql -U airflow
psql (13.2 (Debian 13.2-1.pgdg100+1))
Type "help" for help.

airflow=# \q
root@3058510ae5f7:/# psql -U airflow
psql (13.2 (Debian 13.2-1.pgdg100+1))
Type "help" for help.

airflow=# select * from pg_extension;
  oid  | extname | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition
-------+---------+----------+--------------+----------------+------------+-----------+--------------
 13381 | plpgsql |       10 |           11 | f              | 1.0        |           |
(1 row)

airflow=# CREATE EXTENSION adminpack;
CREATE EXTENSION
airflow=# select * from pg_extension;
  oid  |  extname  | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition
-------+-----------+----------+--------------+----------------+------------+-----------+--------------
 13381 | plpgsql   |       10 |           11 | f              | 1.0        |           |
 16988 | adminpack |       10 |           11 | f              | 2.1        |           |
(2 rows)

airflow=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 airflow   | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```



> Create Function and Trigger

```sql
airflow=# CREATE OR REPLACE FUNCTION ddlogf() RETURNS TRIGGER AS $my_table$
   BEGIN
      PERFORM pg_catalog.pg_file_write('/var/log/postgresql/audit_admin.log', row_to_json(NEW, false)::text || chr(10), true);
      RETURN NEW;
   END;
$my_table$ LANGUAGE plpgsql;
CREATE FUNCTION

airflow=# CREATE TRIGGER ddlogt AFTER INSERT ON log
FOR EACH ROW EXECUTE PROCEDURE ddlogf();
airflow=# \q
```

```bash
root@3058510ae5f7:/# ls -l /var/log/postgresql/
total 8
-rw------- 1 postgres postgres 6675 May 19 18:40 audit_admin.log
```

Mount the new log file to the Kubernetes volume so it can be forwarded to Datadog. The file already contains logs as JSON objects, which should be automatically parsed.

TODO: In the DD postgres pipeline, additional parsing is needed on the `"extra"` JSON key to get: 
* event:paused - ('is_paused', 'false') - place the job in pause (turn DAG off)
* event:paused - ('is_paused', 'true')  - to reactivate the job (turn DAG on) 
* extra:[('confirmed', 'true')] - to get the action of submitting a failed/success from user

### Datadog Integration

> Postgres 10.x+ configuration

```sql
create user datadog with password 'datadog';
grant pg_monitor to datadog;
grant SELECT ON pg_stat_database to datadog;
```


### Airflow Actions & Logs

The following logs are used to create the Datadog dashboard

> Actions when manually interacting with the Airflow interface

```json
click: TREE VIEW
{"id":3856,"dttm":"2021-05-19T23:12:45.593113+00:00","dag_id":"example_bash_operator","task_id":null,"event":"tree","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator')]"}
click: GRAPH VIEW
{"id":3857,"dttm":"2021-05-19T23:13:44.274873+00:00","dag_id":"example_bash_operator","task_id":null,"event":"graph","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator')]"}
click: TASK DURATION
{"id":3858,"dttm":"2021-05-19T23:14:23.479407+00:00","dag_id":"example_bash_operator","task_id":null,"event":"duration","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('days', '30'), ('root', '')]"}
click: TASK TRIES
{"id":3859,"dttm":"2021-05-19T23:15:33.718107+00:00","dag_id":"example_bash_operator","task_id":null,"event":"tries","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('days', '30'), ('root', '')]"}
click: LANDING TIMES 
{"id":3860,"dttm":"2021-05-19T23:15:59.860261+00:00","dag_id":"example_bash_operator","task_id":null,"event":"landing_times","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('days', '30'), ('root', '')]"}
click: GANTT
{"id":3861,"dttm":"2021-05-19T23:16:40.495819+00:00","dag_id":"example_bash_operator","task_id":null,"event":"gantt","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('root', '')]"}
```

> Manually Trigger a DAG (at this point the DAG is not running, it is just hitting the play button)

```json
{"id":3867,"dttm":"2021-05-19T23:25:18.009077+00:00","dag_id":"example_branch_operator","task_id":null,"event":"trigger","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_branch_operator'), ('origin', '/tree?dag_id=example_branch_operator')]"}
```

> Manually trigger a DAG execute/run (hit trigger button)

```json
{"id":3879,"dttm":"2021-05-19T23:53:25.655593+00:00","dag_id":"example_branch_operator","task_id":null,"event":"trigger","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_branch_operator'), ('origin', '/tree?dag_id=example_branch_operator'), ('conf', '{}')]"}
{"id":3880,"dttm":"2021-05-19T23:53:25.740416+00:00","dag_id":"example_branch_operator","task_id":null,"event":"tree","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_branch_operator')]"}
{"id":3881,"dttm":"2021-05-19T23:53:27.692274+00:00","dag_id":"example_branch_operator","task_id":"branching","event":"cli_task_run","execution_date":"2021-05-19T23:53:25.668817+00:00","owner":"airflow","extra":"{\"host_name\": \"4b29006e28b7\", \"full_command\": \"['/home/airflow/.local/bin/airflow', 'celery', 'worker']\"}"}
{"id":3882,"dttm":"2021-05-19T23:53:28.003849+00:00","dag_id":"example_branch_operator","task_id":"branching","event":"running","execution_date":"2021-05-19T23:53:25.668817+00:00","owner":"airflow","extra":null}
{"id":3883,"dttm":"2021-05-19T23:53:28.027881+00:00","dag_id":"example_branch_operator","task_id":"branching","event":"cli_task_run","execution_date":"2021-05-19T23:53:25.668817+00:00","owner":"airflow","extra":"{\"host_name\": \"4b29006e28b7\", \"full_command\": \"['/home/airflow/.local/bin/airflow', 'celery', 'worker']\"}"}
{"id":3884,"dttm":"2021-05-19T23:53:28.185798+00:00","dag_id":"example_branch_operator","task_id":"branching","event":"success","execution_date":"2021-05-19T23:53:25.668817+00:00","owner":"airflow","extra":null}
```


> Manually set a task to Failure

```json
{"id":3987,"dttm":"2021-05-20T00:44:07.609218+00:00","dag_id":"example_bash_operator","task_id":"run_this_last","event":"failed","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('task_id', 'run_this_last'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":3988,"dttm":"2021-05-20T00:44:09.725573+00:00","dag_id":"example_bash_operator","task_id":"run_this_last","event":"failed","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('confirmed', 'true'), ('dag_id', 'example_bash_operator'), ('task_id', 'run_this_last'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":3989,"dttm":"2021-05-20T00:44:09.824443+00:00","dag_id":"example_bash_operator","task_id":null,"event":"tree","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator')]"}
```

> Manually set a task to Success

```json
{"id":3990,"dttm":"2021-05-20T00:45:04.903814+00:00","dag_id":"example_bash_operator","task_id":"run_after_loop","event":"success","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('task_id', 'run_after_loop'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":3991,"dttm":"2021-05-20T00:45:10.043621+00:00","dag_id":"example_bash_operator","task_id":"run_after_loop","event":"success","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('confirmed', 'true'), ('dag_id', 'example_bash_operator'), ('task_id', 'run_after_loop'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":3992,"dttm":"2021-05-20T00:45:10.128239+00:00","dag_id":"example_bash_operator","task_id":null,"event":"tree","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator')]"}
```

> Manually set a DAG as failed

```json
{"id":3998,"dttm":"2021-05-20T00:51:42.855795+00:00","dag_id":"example_bash_operator","task_id":null,"event":"dagrun_failed","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":3999,"dttm":"2021-05-20T00:51:46.129772+00:00","dag_id":"example_bash_operator","task_id":null,"event":"dagrun_failed","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('confirmed', 'true'), ('dag_id', 'example_bash_operator'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":4000,"dttm":"2021-05-20T00:51:46.186286+00:00","dag_id":"example_bash_operator","task_id":null,"event":"tree","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator')]"}
```

> Manually set a DAG as success

```json
{"id":4001,"dttm":"2021-05-20T00:52:42.446402+00:00","dag_id":"example_bash_operator","task_id":null,"event":"dagrun_success","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('dag_id', 'example_bash_operator'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":4002,"dttm":"2021-05-20T00:52:44.487328+00:00","dag_id":"example_bash_operator","task_id":null,"event":"dagrun_success","execution_date":"2021-05-20T00:20:12.427765+00:00","owner":"airflow","extra":"[('confirmed', 'true'), ('dag_id', 'example_bash_operator'), ('execution_date', '2021-05-20T00:20:12.427765+00:00'), ('origin', 'http://192.168.86.37:8080/tree?dag_id=example_bash_operator')]"}
{"id":4003,"dttm":"2021-05-20T00:52:44.692503+00:00","dag_id":"example_bash_operator","task_id":null,"event":"tree","execution_date":null,"owner":"airflow","extra":"[('dag_id', 'example_bash_operator')]"}
```

> Pause a DAG `('is_paused', 'false')`

```json
{"id":4011,"dttm":"2021-05-20T01:10:51.391746+00:00","dag_id":"tutorial","task_id":null,"event":"paused","execution_date":null,"owner":"airflow","extra":"[('is_paused', 'false'), ('dag_id', 'tutorial')]"}
```

> Un-pause DAG `('is_paused', 'true')`

```json
{"id":4012,"dttm":"2021-05-20T01:12:40.427697+00:00","dag_id":"tutorial","task_id":null,"event":"paused","execution_date":null,"owner":"airflow","extra":"[('is_paused', 'true'), ('dag_id', 'tutorial')]"}
```


