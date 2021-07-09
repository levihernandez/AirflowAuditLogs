### Airflow Log Configuration

Datadog provides OOTB configurations to collect Apache Airflow DAG processor manager & DAG Scheduler, this example runs in a host. Airflow config sends all of the DAG triggered tasks to the generic `$HOME_AIRFLOW/logs` directory. 

```bash
$ egrep "log_filename_template" ${HOME_AIRFLOW}/dags/configs/airflow.cfg
log_filename_template = {{ ti.dag_id }}/{{ ti.task_id }}/{{ ts }}/{{ try_number }}.log
```

The logs are generated as follows under `$HOME_AIRFLOW/logs/example_bash_operator/runme_0/2021-04-19T13:23:05.265458+00:00/1.log`, where `example_bash_operator` is the triggered DAG for this example.

> LOG: `Marking task as SUCCESS. dag_id=example_bash_operator, task_id=runme_0, execution_date=20210419T132305, start_date=20210419T132306, end_date=20210419T132308`

The `example_bash_operator` log is generated as follows: 

```bash
AIRFLOW_CTX_DAG_OWNER=airflow
AIRFLOW_CTX_DAG_ID=example_bash_operator
AIRFLOW_CTX_TASK_ID=runme_0
AIRFLOW_CTX_EXECUTION_DATE=2021-04-19T13:23:05.265458+00:00
AIRFLOW_CTX_DAG_RUN_ID=manual__2021-04-19T13:23:05.265458+00:00
[2021-04-19 13:23:06,967] {bash.py:135} INFO - Tmp dir root location: /tmp
[2021-04-19 13:23:06,968] {bash.py:158} INFO - Running command: echo "example_bash_operator__runme_0__20210419" && sleep 1
[2021-04-19 13:23:06,981] {bash.py:169} INFO - Output:
[2021-04-19 13:23:06,983] {bash.py:173} INFO - example_bash_operator__runme_0__20210419
[2021-04-19 13:23:07,985] {bash.py:177} INFO - Command exited with return code 0
[2021-04-19 13:23:08,029] {taskinstance.py:1166} INFO - Marking task as SUCCESS. dag_id=example_bash_operator, task_id=runme_0, execution_date=20210419T132305, start_date=20210419T132306, end_date=20210419T132308
[2021-04-19 13:23:08,068] {taskinstance.py:1220} INFO - 1 downstream tasks scheduled from follow-on schedule check
[2021-04-19 13:23:08,096] {local_task_job.py:146} INFO - Task exited with return code 0
```

### Configure Airflow Triggered DAG Logs

The default log location is configured in the `${HOME_AIRFLOW}/dags/configs/airflow.cfg` through the variable:  
`log_filename_template = {{ ti.dag_id }}/{{ ti.task_id }}/{{ ts }}/{{ try_number }}.log`. Airflow suggests log rotation. However, depending on the volume of tasks scheduled, the default log dir will contain a high volume of dirs corresponding to those tasks. To capture logs by activity such as DAG triggered tasks, it would be better to host all triggered tasks log directories inside a central directory, this helps with the collection of logs by directory name.

> In this example I created a dir called `dag-tasks` to host all of the triggered DAG logs:

```bash
$ mkdir -p ${HOME_AIRFLOW}/dag-tasks
$ vi ${HOME_AIRFLOW}/dags/configs/airflow.cfg
log_filename_template = dag-tasks/{{ ti.dag_id }}/{{ ti.task_id }}/{{ ts }}/{{ try_number }}.log
```

Restart Airflow to apply changes. Next we will register the new log collection with Datadog.

### Datadog Airflow Configuration

The [Datadog Airflow](https://app.datadoghq.com/account/settings#integrations/airflow) integration suggest the capture of the dag_processor_manager.log and scheduler logs. We are adding the triggered DAG logs below:


```yaml
logs:
  - type: file
    path: "<PATH_TO_AIRFLOW>/logs/dag_processor_manager/dag_processor_manager.log"
    source: airflow
    service: "airflow_manager"
    log_processing_rules:
      - type: multi_line
        name: new_log_start_with_date
        pattern: \[\d{4}\-\d{2}\-\d{2}
  - type: file
    path: "<PATH_TO_AIRFLOW>/logs/scheduler/*/*.log"
    source: airflow
    service: "airflow_scheduler"
    log_processing_rules:
      - type: multi_line
        name: new_log_start_with_date
        pattern: \[\d{4}\-\d{2}\-\d{2}
  - type: file
    path: "<PATH_TO_AIRFLOW>/logs/dag-tasks/*/*/*/*.log"
    source: airflow
    service: "airflow_triggered_tasks"
    log_processing_rules:
      - type: multi_line
        name: new_log_start_with_date
        pattern: \[\d{4}\-\d{2}\-\d{2}
```


### Log Patterns - Datadog Pipeline

> Parse the Task log

`Marking task as SUCCESS. dag_id=example_bash_operator, task_id=run_after_loop, execution_date=20210419T171919, start_date=20210419T171923, end_date=20210419T171924`

> Set a Datadog ingestion pipeline parser

`autoFilledRule1 Marking\s+task\s+as\s+%{word:dag_status}.\s+%{data::keyvalue("=","*,")}`
```json
{
  "dag_status": "SUCCESS",
  "dag_id": "example_bash_operator",
  "task_id": "run_after_loop",
  "execution_date": "20210419T171919",
  "start_date": "20210419T171923",
  "end_date": "20210419T171924"
}
```

> Parse the Task log

`Running: ['airflow', 'tasks', 'run', 'example_bash_operator', '*', '2021-04-19T17:19:19.415224+00:00', '--job-id', '[171]', '--pool', 'default_pool', '--raw', '--subdir', '/home/airflow/.local/lib/python3.6/site-packages/airflow/example_dags/example_bash_operator.py', '--cfg-path', '/tmp/* '--error-file', '/tmp/*]`

> Set a Datadog ingestion pipeline parser

`autoFilledRule2 Running\:\s+\[\'%{word:process}\',\s+\'%{word:task_type}\',\s+\'%{word:dag_exec}\',\s+\'%{word:dag_id}\',\s+\'\*\',\s+\'%{date("yyyy-MM-dd'T'HH:mm:ss.SSSSSSZZ"):date}\',\s+\'--job-id\',\s+\'\[%{number:job_id}\]\', '--pool', '%{data:pool}',\s+\'--raw\',\s+\'--subdir\',\s+\'%{data:dag_file}\',\s+\'--cfg-path\',\s+\'/tmp/\*\s+\'--error-file\',\s+\'/tmp/\*\]`

```json
{
  "pool": "default_pool",
  "dag_file": "/home/airflow/.local/lib/python3.6/site-packages/airflow/example_dags/example_bash_operator.py",
  "process": "airflow",
  "task_type": "tasks",
  "dag_exec": "run",
  "dag_id": "example_bash_operator",
  "date": 1618852759415,
  "job_id": 171
}
```

After logs are parsed and show up in the log explorer, create facets on the tags you need. 

```json
{
dag_id
dag_status
end_date
execution_date
start_date
task_id
timestamp
}
```
