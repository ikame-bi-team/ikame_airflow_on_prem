# Set up Airflow on-premise

© 2022 DungNK@iKame.vn

> Make sure you are running as user “airflow” and NOT “root”.

> This guide is for Airflow 2.3.2 only, so make sure you do ```pip install "apache-airflow=2.3.2"```

Features:

- CeleryKubernetesExecutor
- PostgreSQL as Backend DB
- Redis as Message Bus
- MicroK8s as local K8s cluster
- Airflow is installed locally and talks to our local microk8s cluster via kube-config
- Email notifications on failure using SendGrid
- Remote logging to GCS

# I. Set up Linux environment

Update your Linux enviroment with the latest packages. We are using Ubuntu in this deployment.

```
sudo apt update && upgrade -Y
```

## OPTIONAL - Set up VS Code workspace

We are going to use VS Code to easily edit config files. Install extensions that will help you have a more comfortable experience with VPS.

Install these extensions onto the remote VPS:

```
Python
Git
Office Viewer
Path Intellisense
GitHub
```

These extensions enable Python language support, pull/merge/commit with Git, and so on.

# II. Set up python & pip

Run the following command to install pip onto our VPS:

```
sudo apt install python3-pip
```

## Install python libraries required by our DAGs

Libraries:

```
pandas
numpy
requests
google-cloud-bigquery
google-api-python-client
oauth2client
google-cloud-storage
google-analytics-data
pandas-gbq
```

Create a requirements.txt file, with contents are the libraries above. Run:

```
pip install -r requirements.txt
```

# III. Install and set up Airflow and its dependencies

As we are using CeleryKubernetesExecutor, we need to set up both Celery and Kubernetes. First, install Airflow.

## Install Airflow and its dependencies

Run these commands:

```
sudo apt install libmysqlclient-dev
sudo apt install libssl-dev
sudo apt install libkrb5-dev
pip install apache-airflow[celery,google,sendgrid,postgres,redis,cncf.kubernetes]
```

> celery: Enables CeleryExecutor

> google: Enables Google support and installs common Google Python libraries

> sendgrid: Enables email notifications when our DAGs fail

> postgres: Enables postgreSQL as our Backend DB

> redis: Enables redis as our Message Bus

> cncf.kubernetes: Enables KubernetesExecutor

For our DAGs to work correctly, make sure to install PyJWT==2.4.0 by running:
```
pip install "PyJWT==2.4.0"
```

Ignore the warning lights, as Airflow can run just fine.

## Set up postgreSQL

Add the repository that provides PostgreSQL 14 on Ubuntu:

```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

Next import the GPG signing key for the repository.

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

Then update your APT package list.

```
sudo apt -y update
```

Run:

```
sudo apt -y install postgresql-14 postgresql-contrib
```

Ensure that the service is started:

```
sudo systemctl start postgresql.service
```

Install psycopg2:

```
pip install psycopg2-binary
```

Create DB and user inside postgreSQL:

```
sudo -i -u postgres

psql

CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
```

Exit the psql console and log out of the user "postgres" by doing a double "exit" command.

Config pg_hba.conf to accept all incoming connections. This allows our pods to talk to our postgres Backend DB. Run:

```
sudo nano /etc/postgresql/12/main/pg_hba.conf
```

Add this line to the end of the config file:

```
host    all             all             0.0.0.0/0               md5
```

Save and exit.

Config postgresql.conf to listen to all addresses. Run:

```
sudo nano /etc/postgresql/12/main/postgresql.conf
```

Find section "CONNECTIONS AND AUTHENTICATION" - "Connection Settings" - listen_addresses and modify:

```
listen_addresses = '*'
```

Restart postgres to apply changes:

```
sudo systemctl restart postgresql.service
```

## Set up Redis

Install redis-client for Airflow:

```
pip install redis
```

Install redis-server for Airflow:

```
sudo apt install redis-server
```

Next, do:

```
sudo nano /etc/redis/redis.conf
```

To make sure our pods can communicate with Redis, edit the following line:

```
# IF YOU ARE SURE YOU WANT YOUR INSTANCE TO LISTEN TO ALL THE INTERFACES
# JUST COMMENT THE FOLLOWING LINE.
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bind 0.0.0.0

protected-mode no
```

Inside the file, find the supervised directive. This directive allows you to declare an init system to manage Redis as a service, providing you with more control over its operation. The supervised directive is set to no by default. Since you are running Ubuntu, which uses the systemd init system, change this to systemd:

```
. . .

# If you run Redis from upstart or systemd, Redis can interact with your
# supervision tree. Options:
#   supervised no      - no supervision interaction
#   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
#   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
#   supervised auto    - detect upstart or systemd method based on
#                        UPSTART_JOB or NOTIFY_SOCKET environment variables
# Note: these supervision methods only signal "process is ready."
#       They do not enable continuous liveness pings back to your supervisor.
supervised systemd

. . .
```

Then, restart the Redis service to reflect the changes you made to the configuration file:

```
sudo systemctl restart redis.service
```

Start by checking that the Redis service is running:

```
sudo systemctl status redis
```

If it is running without any errors, this command will produce output similar to the following:

```
Output
● redis-server.service - Advanced key-value store
   Loaded: loaded (/lib/systemd/system/redis-server.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2018-06-27 18:48:52 UTC; 12s ago
     Docs: http://redis.io/documentation,
           man:redis-server(1)
  Process: 2421 ExecStop=/bin/kill -s TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 2424 ExecStart=/usr/bin/redis-server /etc/redis/redis.conf (code=exited, status=0/SUCCESS)
 Main PID: 2445 (redis-server)
    Tasks: 4 (limit: 4704)
   CGroup: /system.slice/redis-server.service
           └─2445 /usr/bin/redis-server 127.0.0.1:6379
. . .
```

To test that Redis is functioning correctly, connect to the server using the command-line client:

```
redis-cli
```

In the prompt that follows, test connectivity with the ping command:

```
127.0.0.1:6379 > ping

Output
PONG
```

This output confirms that the server connection is still alive. Next, check that you’re able to set keys by running:

```
set test "It's working!"

Output
OK
```

Retrieve the value by typing:

```
127.0.0.1:6379 > get test

Output
"It's working!"
```

After confirming that you can fetch the value, exit the Redis prompt to get back to the shell:

```
127.0.0.1:6379 > exit
```

As a final test, we will check whether Redis is able to persist data even after it’s been stopped or restarted. To do this, first restart the Redis instance:

```
sudo systemctl restart redis
```

Then connect with the command-line client once again and confirm that your test value is still available:

```
redis-cli
```

```
127.0.0.1:6379 > get test
```

The value of your key should still be accessible:

```
Output
"It's working!"
```

Exit out into the shell again when you are finished:

```
exit
```

With that, your Redis installation is fully operational and ready for you to use with Airflow.

## Set up Airflow

First we set up our Backend DB, then we initialise Airflow.

Create systemd services to manage our Airflow services (webserver, scheduler, flower, celery worker):

> airflow-webserver.service:

```
sudo nano /etc/systemd/system/airflow-webserver.service
```

```
[Unit]
Description=Airflow - Webserver
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
ExecStart=/home/airflow/.local/bin/airflow webserver -p 1234
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

> airflow-scheduler.service:

```
sudo nano /etc/systemd/system/airflow-scheduler.service
```

```
[Unit]
Description=Airflow - Scheduler
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
ExecStart=/home/airflow/.local/bin/airflow scheduler
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

> airflow-flower.service:

```
sudo nano /etc/systemd/system/airflow-flower.service
```

```
[Unit]
Description=Airflow - Celery Flower
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
ExecStart=/home/airflow/.local/bin/airflow celery flower
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

> airflow-worker.service:

```
sudo nano /etc/systemd/system/airflow-worker.service
```

```
[Unit]
Description=Airflow - Scheduler
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
ExecStart=/home/airflow/.local/bin/airflow celery worker
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

In order for these systemd services to function, we need to create an Airflow EnvironmentFile.

```
sudo mkdir /etc/sysconfig/ && nano /etc/sysconfig/airflow
```

Paste these configs into the editor:

```
AIRFLOW_CONFIG=/home/airflow/airflow/airflow.cfg
AIRFLOW_HOME=/home/airflow/airflow
```

Save the file.

Next up, fire up the airflow-webserver.service and airflow-scheduler.service:

```
sudo systemctl start airflow-webserver airflow-scheduler
```

This will create an "Airflow" directory in the HOME folder of our "airflow" user, containing an "airflow.cfg" file that we will use next.

To make sure our Airflow services auto-restart on VPS reboot, do:

```
sudo systemctl enable airflow-webserver airflow-scheduler airflow-flower airflow-worker
```

Edit the "airflow.cfg" config file as follows:

```
default_timezone = Asia/Ho_Chi_Minh
executor = CeleryKubernetesExecutor
enable_xcom_pickling = True
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@10.0.1.1/airflow
remote_logging = True
remote_log_conn_id = google_cloud_default
google_key_path = /home/airflow/bi-team-admin-secrets.json
remote_base_log_folder = gs://airflow-vps-logging-bi-team/logs
google_key_path = /home/airflow/bi-team-admin-secrets.json
default_ui_timezone = Asia/Ho_Chi_Minh
email_backend = airflow.providers.sendgrid.utils.emailer.send_email
email_conn_id = sendgrid_default
from_email = botNKD <DungNK@ikame.vn>
broker_url = redis://10.0.1.1:6379/0
result_backend = db+postgresql://airflow:airflow@10.0.1.1/airflow
flower_url_prefix = /flower
pod_template_file = /home/airflow/pod_template_file.yaml
worker_container_repository = apache/airflow
worker_container_tag = latest-python3.10
namespace = airflow
delete_worker_pods = True
in_cluster = False
cluster_context = microk8s
config_file = /home/airflow/kube-config
```

The final airflow.cfg should look like this:

```
# Airflow.cfg references
# https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html

[core]
# The folder where your airflow pipelines live, most likely a
# subfolder in a code repository. This path must be absolute.
# AIRFLOW__CORE__DAGS_FOLDER
dags_folder = /home/airflow/airflow/dags

# Hostname by providing a path to a callable, which will resolve the hostname.
# The format is "package.function".
#
# For example, default value "socket.getfqdn" means that result from getfqdn() of "socket"
# package will be used as hostname.
#
# No argument should be required in the function specified.
# If using IP address as hostname is preferred, use value ``airflow.utils.net.get_host_ip_address``
hostname_callable = socket.getfqdn

# Default timezone in case supplied date times are naive
# can be utc (default), system, or any IANA timezone string (e.g. Europe/Amsterdam)
# AIRFLOW__CORE__DEFAULT_TIMEZONE
default_timezone = Asia/Ho_Chi_Minh

# The executor class that airflow should use. Choices include
# ``SequentialExecutor``, ``LocalExecutor``, ``CeleryExecutor``, ``DaskExecutor``,
# ``KubernetesExecutor``, ``CeleryKubernetesExecutor`` or the
# full import path to the class when using a custom executor.
executor = CeleryKubernetesExecutor

# This defines the maximum number of task instances that can run concurrently in Airflow
# regardless of scheduler count and worker count. Generally, this value is reflective of
# the number of task instances with the running state in the metadata database.
parallelism = 32

# The maximum number of task instances allowed to run concurrently in each DAG. To calculate
# the number of tasks that is running concurrently for a DAG, add up the number of running
# tasks for all DAG runs of the DAG. This is configurable at the DAG level with ``max_active_tasks``,
# which is defaulted as ``max_active_tasks_per_dag``.
#
# An example scenario when this would be useful is when you want to stop a new dag with an early
# start date from stealing all the executor slots in a cluster.
max_active_tasks_per_dag = 8

# Are DAGs paused by default at creation
dags_are_paused_at_creation = True

# The maximum number of active DAG runs per DAG. The scheduler will not create more DAG runs
# if it reaches the limit. This is configurable at the DAG level with ``max_active_runs``,
# which is defaulted as ``max_active_runs_per_dag``.
max_active_runs_per_dag = 1

# Whether to load the DAG examples that ship with Airflow. It's good to
# get started, but you probably want to set this to ``False`` in a production
# environment
load_examples = False

# Path to the folder containing Airflow plugins
plugins_folder = /home/airflow/airflow/plugins

# Should tasks be executed via forking of the parent process ("False",
# the speedier option) or by spawning a new python process ("True" slow,
# but means plugin changes picked up by tasks straight away)
execute_tasks_new_python_interpreter = False

# Secret key to save connection passwords in the db
fernet_key = ************************************

# Whether to disable pickling dags
donot_pickle = True

# How long before timing out a python file import
dagbag_import_timeout = 30.0

# Should a traceback be shown in the UI for dagbag import errors,
# instead of just the exception message
dagbag_import_error_tracebacks = True

# If tracebacks are shown, how many entries from the traceback should be shown
dagbag_import_error_traceback_depth = 2

# How long before timing out a DagFileProcessor, which processes a dag file
dag_file_processor_timeout = 50

# The class to use for running task instances in a subprocess.
# Choices include StandardTaskRunner, CgroupTaskRunner or the full import path to the class
# when using a custom task runner.
task_runner = StandardTaskRunner

# If set, tasks without a ``run_as_user`` argument will be run with this user
# Can be used to de-elevate a sudo user running Airflow when executing tasks
default_impersonation =

# What security module to use (for example kerberos)
security =

# Turn unit test mode on (overwrites many configuration options with test
# values at runtime)
unit_test_mode = False

# Whether to enable pickling for xcom (note that this is insecure and allows for
# RCE exploits).
# AIRFLOW__CORE__ENABLE_XCOM_PICKLING
enable_xcom_pickling = True

# When a task is killed forcefully, this is the amount of time in seconds that
# it has to cleanup after it is sent a SIGTERM, before it is SIGKILLED
killed_task_cleanup_time = 60

# Whether to override params with dag_run.conf. If you pass some key-value pairs
# through ``airflow dags backfill -c`` or
# ``airflow dags trigger -c``, the key-value pairs will override the existing ones in params.
dag_run_conf_overrides_params = True

# When discovering DAGs, ignore any files that don't contain the strings ``DAG`` and ``airflow``.
dag_discovery_safe_mode = True

# The pattern syntax used in the ".airflowignore" files in the DAG directories. Valid values are
# ``regexp`` or ``glob``.
dag_ignore_file_syntax = regexp

# The number of retries each task is going to have by default. Can be overridden at dag or task level.
default_task_retries = 0

# The weighting method used for the effective total priority weight of the task
default_task_weight_rule = downstream

# The default task execution_timeout value for the operators. Expected an integer value to
# be passed into timedelta as seconds. If not specified, then the value is considered as None,
# meaning that the operators are never timed out by default.
default_task_execution_timeout =

# Updating serialized DAG can not be faster than a minimum interval to reduce database write rate.
min_serialized_dag_update_interval = 30

# If True, serialized DAGs are compressed before writing to DB.
# Note: this will disable the DAG dependencies view
compress_serialized_dags = False

# Fetching serialized DAG can not be faster than a minimum interval to reduce database
# read rate. This config controls when your DAGs are updated in the Webserver
min_serialized_dag_fetch_interval = 10

# Maximum number of Rendered Task Instance Fields (Template Fields) per task to store
# in the Database.
# All the template_fields for each of Task Instance are stored in the Database.
# Keeping this number small may cause an error when you try to view ``Rendered`` tab in
# TaskInstance view for older tasks.
max_num_rendered_ti_fields_per_task = 30

# On each dagrun check against defined SLAs
check_slas = True

# Path to custom XCom class that will be used to store and resolve operators results
# Example: xcom_backend = path.to.CustomXCom
xcom_backend = airflow.models.xcom.BaseXCom

# By default Airflow plugins are lazily-loaded (only loaded when required). Set it to ``False``,
# if you want to load plugins whenever 'airflow' is invoked via cli or loaded from module.
lazy_load_plugins = True

# By default Airflow providers are lazily-discovered (discovery and imports happen only when required).
# Set it to False, if you want to discover providers whenever 'airflow' is invoked via cli or
# loaded from module.
lazy_discover_providers = True

# Hide sensitive Variables or Connection extra json keys from UI and task logs when set to True
#
# (Connection passwords are always hidden in logs)
hide_sensitive_var_conn_fields = True

# A comma-separated list of extra sensitive keywords to look for in variables names or connection's
# extra JSON.
sensitive_var_conn_names =

# Task Slot counts for ``default_pool``. This setting would not have any effect in an existing
# deployment where the ``default_pool`` is already created. For existing deployments, users can
# change the number of slots using Webserver, API or the CLI
default_pool_task_slot_count = 128

# The maximum list/dict length an XCom can push to trigger task mapping. If the pushed list/dict has a
# length exceeding this value, the task pushing the XCom will be failed automatically to prevent the
# mapped tasks from clogging the scheduler.
max_map_length = 1024

[database]
# The SqlAlchemy connection string to the metadata database.
# SqlAlchemy supports many different database engines.
# More information here:
# http://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html#database-uri
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@10.0.1.1/airflow

# Extra engine specific keyword args passed to SQLAlchemy's create_engine, as a JSON-encoded value
# Example: sql_alchemy_engine_args = {"arg1": True}
# sql_alchemy_engine_args =

# The encoding for the databases
sql_engine_encoding = utf-8

# Collation for ``dag_id``, ``task_id``, ``key`` columns in case they have different encoding.
# By default this collation is the same as the database collation, however for ``mysql`` and ``mariadb``
# the default is ``utf8mb3_bin`` so that the index sizes of our index keys will not exceed
# the maximum size of allowed index when collation is set to ``utf8mb4`` variant
# (see https://github.com/apache/airflow/pull/17603#issuecomment-901121618).
# sql_engine_collation_for_ids =

# If SqlAlchemy should pool database connections.
sql_alchemy_pool_enabled = True

# The SqlAlchemy pool size is the maximum number of database connections
# in the pool. 0 indicates no limit.
sql_alchemy_pool_size = 5

# The maximum overflow size of the pool.
# When the number of checked-out connections reaches the size set in pool_size,
# additional connections will be returned up to this limit.
# When those additional connections are returned to the pool, they are disconnected and discarded.
# It follows then that the total number of simultaneous connections the pool will allow
# is pool_size + max_overflow,
# and the total number of "sleeping" connections the pool will allow is pool_size.
# max_overflow can be set to ``-1`` to indicate no overflow limit;
# no limit will be placed on the total number of concurrent connections. Defaults to ``10``.
sql_alchemy_max_overflow = 10

# The SqlAlchemy pool recycle is the number of seconds a connection
# can be idle in the pool before it is invalidated. This config does
# not apply to sqlite. If the number of DB connections is ever exceeded,
# a lower config value will allow the system to recover faster.
sql_alchemy_pool_recycle = 1800

# Check connection at the start of each connection pool checkout.
# Typically, this is a simple statement like "SELECT 1".
# More information here:
# https://docs.sqlalchemy.org/en/13/core/pooling.html#disconnect-handling-pessimistic
sql_alchemy_pool_pre_ping = True

# The schema to use for the metadata database.
# SqlAlchemy supports databases with the concept of multiple schemas.
sql_alchemy_schema =

# Import path for connect args in SqlAlchemy. Defaults to an empty dict.
# This is useful when you want to configure db engine args that SqlAlchemy won't parse
# in connection string.
# See https://docs.sqlalchemy.org/en/13/core/engines.html#sqlalchemy.create_engine.params.connect_args
# sql_alchemy_connect_args =

# Whether to load the default connections that ship with Airflow. It's good to
# get started, but you probably want to set this to ``False`` in a production
# environment
load_default_connections = True

# Number of times the code should be retried in case of DB Operational Errors.
# Not all transactions will be retried as it can cause undesired state.
# Currently it is only used in ``DagFileProcessor.process_file`` to retry ``dagbag.sync_to_db``.
max_db_retries = 3

[logging]
# The folder where airflow should store its log files.
# This path must be absolute.
# There are a few existing configurations that assume this is set to the default.
# If you choose to override this you may need to update the dag_processor_manager_log_location and
# dag_processor_manager_log_location settings as well.
base_log_folder = /home/airflow/airflow/logs

# Airflow can store logs remotely in AWS S3, Google Cloud Storage or Elastic Search.
# Set this to True if you want to enable remote logging.
remote_logging = True

# Users must supply an Airflow connection id that provides access to the storage
# location. Depending on your remote logging service, this may only be used for
# reading logs, not writing them.
remote_log_conn_id = google_cloud_default

# Path to Google Credential JSON file. If omitted, authorization based on `the Application Default
# Credentials
# <https://cloud.google.com/docs/authentication/production#finding_credentials_automatically>`__ will
# be used.
google_key_path = /home/airflow/bi-team-admin-secrets.json

# Storage bucket URL for remote logging
# S3 buckets should start with "s3://"
# Cloudwatch log groups should start with "cloudwatch://"
# GCS buckets should start with "gs://"
# WASB buckets should start with "wasb" just to help Airflow select correct handler
# Stackdriver logs should start with "stackdriver://"
remote_base_log_folder = gs://airflow-vps-logging-bi-team/logs

# Use server-side encryption for logs stored in S3
encrypt_s3_logs = False

# Logging level.
#
# Supported values: ``CRITICAL``, ``ERROR``, ``WARNING``, ``INFO``, ``DEBUG``.
logging_level = INFO

# Logging level for celery. If not set, it uses the value of logging_level
#
# Supported values: ``CRITICAL``, ``ERROR``, ``WARNING``, ``INFO``, ``DEBUG``.
celery_logging_level =

# Logging level for Flask-appbuilder UI.
#
# Supported values: ``CRITICAL``, ``ERROR``, ``WARNING``, ``INFO``, ``DEBUG``.
fab_logging_level = WARNING

# Logging class
# Specify the class that will specify the logging configuration
# This class has to be on the python classpath
# Example: logging_config_class = my.path.default_local_settings.LOGGING_CONFIG
logging_config_class =

# Flag to enable/disable Colored logs in Console
# Colour the logs when the controlling terminal is a TTY.
colored_console_log = True

# Log format for when Colored logs is enabled
colored_log_format = [%%(blue)s%%(asctime)s%%(reset)s] {%%(blue)s%%(filename)s:%%(reset)s%%(lineno)d} %%(log_color)s%%(levelname)s%%(reset)s - %%(log_color)s%%(message)s%%(reset)s
colored_formatter_class = airflow.utils.log.colored_log.CustomTTYColoredFormatter

# Format of Log line
log_format = [%%(asctime)s] {%%(filename)s:%%(lineno)d} %%(levelname)s - %%(message)s
simple_log_format = %%(asctime)s %%(levelname)s - %%(message)s

# Specify prefix pattern like mentioned below with stream handler TaskHandlerWithCustomFormatter
# Example: task_log_prefix_template = {ti.dag_id}-{ti.task_id}-{execution_date}-{try_number}
task_log_prefix_template =

# Formatting for how airflow generates file names/paths for each task run.
log_filename_template = dag_id={{ ti.dag_id }}/run_id={{ ti.run_id }}/task_id={{ ti.task_id }}/{%% if ti.map_index >= 0 %%}map_index={{ ti.map_index }}/{%% endif %%}attempt={{ try_number }}.log

# Formatting for how airflow generates file names for log
log_processor_filename_template = {{ filename }}.log

# Full path of dag_processor_manager logfile.
dag_processor_manager_log_location = /home/airflow/airflow/logs/dag_processor_manager/dag_processor_manager.log

# Name of handler to read task instance logs.
# Defaults to use ``task`` handler.
task_log_reader = task

# A comma\-separated list of third-party logger names that will be configured to print messages to
# consoles\.
# Example: extra_logger_names = connexion,sqlalchemy
extra_logger_names =

# When you start an airflow worker, airflow starts a tiny web server
# subprocess to serve the workers local log files to the airflow main
# web server, who then builds pages and sends them to users. This defines
# the port on which the logs are served. It needs to be unused, and open
# visible from the main web server to connect into the workers.
worker_log_server_port = 8793

[metrics]

# StatsD (https://github.com/etsy/statsd) integration settings.
# Enables sending metrics to StatsD.
statsd_on = False
statsd_host = localhost
statsd_port = 8125
statsd_prefix = airflow

# If you want to avoid sending all the available metrics to StatsD,
# you can configure an allow list of prefixes (comma separated) to send only the metrics that
# start with the elements of the list (e.g: "scheduler,executor,dagrun")
statsd_allow_list =

# A function that validate the StatsD stat name, apply changes to the stat name if necessary and return
# the transformed stat name.
#
# The function should have the following signature:
# def func_name(stat_name: str) -> str:
stat_name_handler =

# To enable datadog integration to send airflow metrics.
statsd_datadog_enabled = False

# List of datadog tags attached to all metrics(e.g: key1:value1,key2:value2)
statsd_datadog_tags =

# If you want to utilise your own custom StatsD client set the relevant
# module path below.
# Note: The module path must exist on your PYTHONPATH for Airflow to pick it up
# statsd_custom_client_path =

[secrets]
# Full class name of secrets backend to enable (will precede env vars and metastore in search path)
# Example: backend = airflow.providers.amazon.aws.secrets.systems_manager.SystemsManagerParameterStoreBackend
backend =

# The backend_kwargs param is loaded into a dictionary and passed to __init__ of secrets backend class.
# See documentation for the secrets backend you are using. JSON is expected.
# Example for AWS Systems Manager ParameterStore:
# ``{"connections_prefix": "/airflow/connections", "profile_name": "default"}``
backend_kwargs =

[cli]
# In what way should the cli access the API. The LocalClient will use the
# database directly, while the json_client will use the api running on the
# webserver
api_client = airflow.api.client.local_client

# If you set web_server_url_prefix, do NOT forget to append it here, ex:
# ``endpoint_url = http://localhost:8080/myroot``
# So api will look like: ``http://localhost:8080/myroot/api/experimental/...``
endpoint_url = http://localhost:8080

[debug]
# Used only with ``DebugExecutor``. If set to ``True`` DAG will fail with first
# failed task. Helpful for debugging purposes.
fail_fast = False

[api]
# Enables the deprecated experimental API. Please note that these APIs do not have access control.
# The authenticated user has full access.
#
# .. warning::
#
#   This `Experimental REST API <https://airflow.readthedocs.io/en/latest/rest-api-ref.html>`__ is
#   deprecated since version 2.0. Please consider using
#   `the Stable REST API <https://airflow.readthedocs.io/en/latest/stable-rest-api-ref.html>`__.
#   For more information on migration, see
#   `RELEASE_NOTES.rst <https://github.com/apache/airflow/blob/main/RELEASE_NOTES.rst>`_
enable_experimental_api = False

# Comma separated list of auth backends to authenticate users of the API. See
# https://airflow.apache.org/docs/apache-airflow/stable/security/api.html for possible values.
# ("airflow.api.auth.backend.default" allows all requests for historic reasons)
auth_backends = airflow.api.auth.backend.session

# Used to set the maximum page limit for API requests
maximum_page_limit = 100

# Used to set the default page limit when limit is zero. A default limit
# of 100 is set on OpenApi spec. However, this particular default limit
# only work when limit is set equal to zero(0) from API requests.
# If no limit is supplied, the OpenApi spec default is used.
fallback_page_limit = 100

# The intended audience for JWT token credentials used for authorization. This value must match on the client and server sides. If empty, audience will not be tested.
# Example: google_oauth2_audience = project-id-random-value.apps.googleusercontent.com
google_oauth2_audience =

# Path to Google Cloud Service Account key file (JSON). If omitted, authorization based on
# `the Application Default Credentials
# <https://cloud.google.com/docs/authentication/production#finding_credentials_automatically>`__ will
# be used.
# Example: google_key_path = /files/service-account-json
google_key_path = /home/airflow/bi-team-admin-secrets.json

# Used in response to a preflight request to indicate which HTTP
# headers can be used when making the actual request. This header is
# the server side response to the browser's
# Access-Control-Request-Headers header.
access_control_allow_headers =

# Specifies the method or methods allowed when accessing the resource.
access_control_allow_methods =

# Indicates whether the response can be shared with requesting code from the given origins.
# Separate URLs with space.
access_control_allow_origins =

[lineage]
# what lineage backend to use
backend =

[atlas]
sasl_enabled = False
host =
port = 21000
username =
password =

[operators]
# The default owner assigned to each new operator, unless
# provided explicitly or passed via ``default_args``
default_owner = airflow
default_cpus = 1
default_ram = 512
default_disk = 512
default_gpus = 0

# Default queue that tasks get assigned to and that worker listen on.
default_queue = default

# Is allowed to pass additional/unused arguments (args, kwargs) to the BaseOperator operator.
# If set to False, an exception will be thrown, otherwise only the console message will be displayed.
allow_illegal_arguments = False

[hive]
# Default mapreduce queue for HiveOperator tasks
default_hive_mapred_queue =

# Template for mapred_job_name in HiveOperator, supports the following named parameters
# hostname, dag_id, task_id, execution_date
# mapred_job_name_template =

[webserver]
# The base url of your website as airflow cannot guess what domain or
# cname you are using. This is used in automated emails that
# airflow sends to point links to the right web server
base_url = http://localhost:8080

# Default timezone to display all dates in the UI, can be UTC, system, or
# any IANA timezone string (e.g. Europe/Amsterdam). If left empty the
# default value of core/default_timezone will be used
# Example: default_ui_timezone = America/New_York
# AIRFLOW__WEBSERVER__DEFAULT_UI_TIMEZONE
default_ui_timezone = Asia/Ho_Chi_Minh

# The ip specified when starting the web server
web_server_host = 0.0.0.0

# The port on which to run the web server
web_server_port = 8080

# Paths to the SSL certificate and key for the web server. When both are
# provided SSL will be enabled. This does not change the web server port.
web_server_ssl_cert =

# Paths to the SSL certificate and key for the web server. When both are
# provided SSL will be enabled. This does not change the web server port.
web_server_ssl_key =

# The type of backend used to store web session data, can be 'database' or 'securecookie'
# Example: session_backend = securecookie
session_backend = database

# Number of seconds the webserver waits before killing gunicorn master that doesn't respond
web_server_master_timeout = 120

# Number of seconds the gunicorn webserver waits before timing out on a worker
web_server_worker_timeout = 120

# Number of workers to refresh at a time. When set to 0, worker refresh is
# disabled. When nonzero, airflow periodically refreshes webserver workers by
# bringing up new ones and killing old ones.
worker_refresh_batch_size = 1

# Number of seconds to wait before refreshing a batch of workers.
worker_refresh_interval = 6000

# If set to True, Airflow will track files in plugins_folder directory. When it detects changes,
# then reload the gunicorn.
reload_on_plugin_change = False

# Secret key used to run your flask app. It should be as random as possible. However, when running
# more than 1 instances of webserver, make sure all of them use the same ``secret_key`` otherwise
# one of them will error with "CSRF session token is missing".
# The webserver key is also used to authorize requests to Celery workers when logs are retrieved.
# The token generated using the secret key has a short expiry time though - make sure that time on
# ALL the machines that you run airflow components on is synchronized (for example using ntpd)
# otherwise you might get "forbidden" errors when the logs are accessed.
secret_key = JehsOymLrb9BEVguE0Cabg==

# Number of workers to run the Gunicorn web server
workers = 4

# The worker class gunicorn should use. Choices include
# sync (default), eventlet, gevent
worker_class = sync

# Log files for the gunicorn webserver. '-' means log to stderr.
access_logfile = -

# Log files for the gunicorn webserver. '-' means log to stderr.
error_logfile = -

# Access log format for gunicorn webserver.
# default format is %%(h)s %%(l)s %%(u)s %%(t)s "%%(r)s" %%(s)s %%(b)s "%%(f)s" "%%(a)s"
# documentation - https://docs.gunicorn.org/en/stable/settings.html#access-log-format
access_logformat =

# Expose the configuration file in the web server
expose_config = True

# Expose hostname in the web server
expose_hostname = True

# Expose stacktrace in the web server
expose_stacktrace = True

# Default DAG view. Valid values are: ``grid``, ``graph``, ``duration``, ``gantt``, ``landing_times``
dag_default_view = grid

# Default DAG orientation. Valid values are:
# ``LR`` (Left->Right), ``TB`` (Top->Bottom), ``RL`` (Right->Left), ``BT`` (Bottom->Top)
dag_orientation = LR

# The amount of time (in secs) webserver will wait for initial handshake
# while fetching logs from other worker machine
log_fetch_timeout_sec = 5

# Time interval (in secs) to wait before next log fetching.
log_fetch_delay_sec = 2

# Distance away from page bottom to enable auto tailing.
log_auto_tailing_offset = 30

# Animation speed for auto tailing log display.
log_animation_speed = 1000

# By default, the webserver shows paused DAGs. Flip this to hide paused
# DAGs by default
hide_paused_dags_by_default = False

# Consistent page size across all listing views in the UI
page_size = 100

# Define the color of navigation bar
navbar_color = #fff

# Default dagrun to show in UI
default_dag_run_display_number = 25

# Enable werkzeug ``ProxyFix`` middleware for reverse proxy
enable_proxy_fix = False

# Number of values to trust for ``X-Forwarded-For``.
# More info: https://werkzeug.palletsprojects.com/en/0.16.x/middleware/proxy_fix/
proxy_fix_x_for = 1

# Number of values to trust for ``X-Forwarded-Proto``
proxy_fix_x_proto = 1

# Number of values to trust for ``X-Forwarded-Host``
proxy_fix_x_host = 1

# Number of values to trust for ``X-Forwarded-Port``
proxy_fix_x_port = 1

# Number of values to trust for ``X-Forwarded-Prefix``
proxy_fix_x_prefix = 1

# Set secure flag on session cookie
cookie_secure = False

# Set samesite policy on session cookie
cookie_samesite = Lax

# Default setting for wrap toggle on DAG code and TI log views.
default_wrap = False

# Allow the UI to be rendered in a frame
x_frame_enabled = True

# Send anonymous user activity to your analytics tool
# choose from google_analytics, segment, or metarouter
# analytics_tool =

# Unique ID of your account in the analytics tool
# analytics_id =

# 'Recent Tasks' stats will show for old DagRuns if set
show_recent_stats_for_completed_runs = True

# Update FAB permissions and sync security manager roles
# on webserver startup
update_fab_perms = True

# The UI cookie lifetime in minutes. User will be logged out from UI after
# ``session_lifetime_minutes`` of non-activity
session_lifetime_minutes = 43200

# Sets a custom page title for the DAGs overview page and site title for all pages
# instance_name =

# Whether the custom page title for the DAGs overview page contains any Markup language
instance_name_has_markup = False

# How frequently, in seconds, the DAG data will auto-refresh in graph or grid view
# when auto-refresh is turned on
auto_refresh_interval = 3

# Boolean for displaying warning for publicly viewable deployment
warn_deployment_exposure = False

# Comma separated string of view events to exclude from dag audit view.
# All other events will be added minus the ones passed here.
# The audit logs in the db will not be affected by this parameter.
audit_view_excluded_events = gantt,landing_times,tries,duration,calendar,graph,grid,tree,tree_data

# Comma separated string of view events to include in dag audit view.
# If passed, only these events will populate the dag audit view.
# The audit logs in the db will not be affected by this parameter.
# Example: audit_view_included_events = dagrun_cleared,failed
# audit_view_included_events =

[email]

# Configuration email backend and whether to
# send email alerts on retry or failure
# Email backend to use
email_backend = airflow.providers.sendgrid.utils.emailer.send_email

# Email connection to use
email_conn_id = sendgrid_default

# Whether email alerts should be sent when a task is retried
default_email_on_retry = True

# Whether email alerts should be sent when a task failed
default_email_on_failure = True

# File that will be used as the template for Email subject (which will be rendered using Jinja2).
# If not set, Airflow uses a base template.
# Example: subject_template = /path/to/my_subject_template_file
# subject_template =

# File that will be used as the template for Email content (which will be rendered using Jinja2).
# If not set, Airflow uses a base template.
# Example: html_content_template = /path/to/my_html_content_template_file
# html_content_template =

# Email address that will be used as sender address.
# It can either be raw email or the complete address in a format ``Sender Name <sender@email.com>``
# Example: from_email = Airflow <airflow@example.com>
from_email = botNKD <DungNK@ikame.vn>

[smtp]

# If you want airflow to send emails on retries, failure, and you want to use
# the airflow.utils.email.send_email_smtp function, you have to configure an
# smtp server here
smtp_host = localhost
smtp_starttls = True
smtp_ssl = False
# Example: smtp_user = airflow
# smtp_user =
# Example: smtp_password = airflow
# smtp_password =
smtp_port = 25
smtp_mail_from = airflow@example.com
smtp_timeout = 30
smtp_retry_limit = 5

[sentry]

# Sentry (https://docs.sentry.io) integration. Here you can supply
# additional configuration options based on the Python platform. See:
# https://docs.sentry.io/error-reporting/configuration/?platform=python.
# Unsupported options: ``integrations``, ``in_app_include``, ``in_app_exclude``,
# ``ignore_errors``, ``before_breadcrumb``, ``transport``.
# Enable error reporting to Sentry
sentry_on = false
sentry_dsn =

# Dotted path to a before_send function that the sentry SDK should be configured to use.
# before_send =

[local_kubernetes_executor]

# This section only applies if you are using the ``LocalKubernetesExecutor`` in
# ``[core]`` section above
# Define when to send a task to ``KubernetesExecutor`` when using ``LocalKubernetesExecutor``.
# When the queue of a task is the value of ``kubernetes_queue`` (default ``kubernetes``),
# the task is executed via ``KubernetesExecutor``,
# otherwise via ``LocalExecutor``
kubernetes_queue = kubernetes

[celery_kubernetes_executor]

# This section only applies if you are using the ``CeleryKubernetesExecutor`` in
# ``[core]`` section above
# Define when to send a task to ``KubernetesExecutor`` when using ``CeleryKubernetesExecutor``.
# When the queue of a task is the value of ``kubernetes_queue`` (default ``kubernetes``),
# the task is executed via ``KubernetesExecutor``,
# otherwise via ``CeleryExecutor``
kubernetes_queue = kubernetes

[celery]

# This section only applies if you are using the CeleryExecutor in
# ``[core]`` section above
# The app name that will be used by celery
celery_app_name = airflow.executors.celery_executor

# The concurrency that will be used when starting workers with the
# ``airflow celery worker`` command. This defines the number of task instances that
# a worker will take, so size up your workers based on the resources on
# your worker box and the nature of your tasks
worker_concurrency = 16

# The maximum and minimum concurrency that will be used when starting workers with the
# ``airflow celery worker`` command (always keep minimum processes, but grow
# to maximum if necessary). Note the value should be max_concurrency,min_concurrency
# Pick these numbers based on resources on worker box and the nature of the task.
# If autoscale option is available, worker_concurrency will be ignored.
# http://docs.celeryproject.org/en/latest/reference/celery.bin.worker.html#cmdoption-celery-worker-autoscale
# Example: worker_autoscale = 16,12
worker_autoscale = 16,8

# Used to increase the number of tasks that a worker prefetches which can improve performance.
# The number of processes multiplied by worker_prefetch_multiplier is the number of tasks
# that are prefetched by a worker. A value greater than 1 can result in tasks being unnecessarily
# blocked if there are multiple workers and one worker prefetches tasks that sit behind long
# running tasks while another worker has unutilized processes that are unable to process the already
# claimed blocked tasks.
# https://docs.celeryproject.org/en/stable/userguide/optimizing.html#prefetch-limits
worker_prefetch_multiplier = 1

# Specify if remote control of the workers is enabled.
# When using Amazon SQS as the broker, Celery creates lots of ``.*reply-celery-pidbox`` queues. You can
# prevent this by setting this to false. However, with this disabled Flower won't work.
worker_enable_remote_control = true

# Umask that will be used when starting workers with the ``airflow celery worker``
# in daemon mode. This control the file-creation mode mask which determines the initial
# value of file permission bits for newly created files.
worker_umask = 0o077

# The Celery broker URL. Celery supports RabbitMQ, Redis and experimentally
# a sqlalchemy database. Refer to the Celery documentation for more information.
# AIRFLOW__CELERY__BROKER_URL
broker_url = redis://10.0.1.1:6379/0

# The Celery result_backend. When a job finishes, it needs to update the
# metadata of the job. Therefore it will post a message on a message bus,
# or insert it into a database (depending of the backend)
# This status is used by the scheduler to update the state of the task
# The use of a database is highly recommended
# http://docs.celeryproject.org/en/latest/userguide/configuration.html#task-result-backend-settings
result_backend = db+postgresql://airflow:airflow@10.0.1.1/airflow

# Celery Flower is a sweet UI for Celery. Airflow has a shortcut to start
# it ``airflow celery flower``. This defines the IP that Celery Flower runs on
flower_host = 0.0.0.0

# The root URL for Flower
# Example: flower_url_prefix = /flower
flower_url_prefix = /flower

# This defines the port that Celery Flower runs on
flower_port = 5555

# Securing Flower with Basic Authentication
# Accepts user:password pairs separated by a comma
# Example: flower_basic_auth = user1:password1,user2:password2
flower_basic_auth =

# How many processes CeleryExecutor uses to sync task state.
# 0 means to use max(1, number of cores - 1) processes.
sync_parallelism = 0

# Import path for celery configuration options
celery_config_options = airflow.config_templates.default_celery.DEFAULT_CELERY_CONFIG
ssl_active = False
ssl_key =
ssl_cert =
ssl_cacert =

# Celery Pool implementation.
# Choices include: ``prefork`` (default), ``eventlet``, ``gevent`` or ``solo``.
# See:
# https://docs.celeryproject.org/en/latest/userguide/workers.html#concurrency
# https://docs.celeryproject.org/en/latest/userguide/concurrency/eventlet.html
pool = prefork

# The number of seconds to wait before timing out ``send_task_to_executor`` or
# ``fetch_celery_task_state`` operations.
operation_timeout = 1.0

# Celery task will report its status as 'started' when the task is executed by a worker.
# This is used in Airflow to keep track of the running tasks and if a Scheduler is restarted
# or run in HA mode, it can adopt the orphan tasks launched by previous SchedulerJob.
task_track_started = True

# Time in seconds after which adopted tasks which are queued in celery are assumed to be stalled,
# and are automatically rescheduled. This setting does the same thing as ``stalled_task_timeout`` but
# applies specifically to adopted tasks only. When set to 0, the ``stalled_task_timeout`` setting
# also applies to adopted tasks.
task_adoption_timeout = 600

# Time in seconds after which tasks queued in celery are assumed to be stalled, and are automatically
# rescheduled. Adopted tasks will instead use the ``task_adoption_timeout`` setting if specified.
# When set to 0, automatic clearing of stalled tasks is disabled.
stalled_task_timeout = 0

# The Maximum number of retries for publishing task messages to the broker when failing
# due to ``AirflowTaskTimeout`` error before giving up and marking Task as failed.
task_publish_max_retries = 3

# Worker initialisation check to validate Metadata Database connection
worker_precheck = False

[celery_broker_transport_options]

# This section is for specifying options which can be passed to the
# underlying celery broker transport. See:
# http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-broker_transport_options
# The visibility timeout defines the number of seconds to wait for the worker
# to acknowledge the task before the message is redelivered to another worker.
# Make sure to increase the visibility timeout to match the time of the longest
# ETA you're planning to use.
# visibility_timeout is only supported for Redis and SQS celery brokers.
# See:
# http://docs.celeryproject.org/en/master/userguide/configuration.html#std:setting-broker_transport_options
# Example: visibility_timeout = 21600
# visibility_timeout =

[dask]

# This section only applies if you are using the DaskExecutor in
# [core] section above
# The IP address and port of the Dask cluster's scheduler.
cluster_address = 127.0.0.1:8786

# TLS/ SSL settings to access a secured Dask scheduler.
tls_ca =
tls_cert =
tls_key =

[scheduler]
# Task instances listen for external kill signal (when you clear tasks
# from the CLI or the UI), this defines the frequency at which they should
# listen (in seconds).
job_heartbeat_sec = 5

# The scheduler constantly tries to trigger new tasks (look at the
# scheduler section in the docs for more information). This defines
# how often the scheduler should run (in seconds).
scheduler_heartbeat_sec = 5

# The number of times to try to schedule each DAG file
# -1 indicates unlimited number
num_runs = -1

# Controls how long the scheduler will sleep between loops, but if there was nothing to do
# in the loop. i.e. if it scheduled something then it will start the next loop
# iteration straight away.
scheduler_idle_sleep_time = 1

# Number of seconds after which a DAG file is parsed. The DAG file is parsed every
# ``min_file_process_interval`` number of seconds. Updates to DAGs are reflected after
# this interval. Keeping this number low will increase CPU usage.
min_file_process_interval = 15

# How often (in seconds) to check for stale DAGs (DAGs which are no longer present in
# the expected files) which should be deactivated.
deactivate_stale_dags_interval = 60

# How often (in seconds) to scan the DAGs directory for new files. Default to 5 minutes.
dag_dir_list_interval = 30

# How often should stats be printed to the logs. Setting to 0 will disable printing stats
print_stats_interval = 5

# How often (in seconds) should pool usage stats be sent to StatsD (if statsd_on is enabled)
pool_metrics_interval = 5.0

# If the last scheduler heartbeat happened more than scheduler_health_check_threshold
# ago (in seconds), scheduler is considered unhealthy.
# This is used by the health check in the "/health" endpoint
scheduler_health_check_threshold = 30

# How often (in seconds) should the scheduler check for orphaned tasks and SchedulerJobs
orphaned_tasks_check_interval = 300.0
child_process_log_directory = /home/airflow/airflow/logs/scheduler

# Local task jobs periodically heartbeat to the DB. If the job has
# not heartbeat in this many seconds, the scheduler will mark the
# associated task instance as failed and will re-schedule the task.
scheduler_zombie_task_threshold = 300

# How often (in seconds) should the scheduler check for zombie tasks.
zombie_detection_interval = 10.0

# Turn off scheduler catchup by setting this to ``False``.
# Default behavior is unchanged and
# Command Line Backfills still work, but the scheduler
# will not do scheduler catchup if this is ``False``,
# however it can be set on a per DAG basis in the
# DAG definition (catchup)
catchup_by_default = True

# Setting this to True will make first task instance of a task
# ignore depends_on_past setting. A task instance will be considered
# as the first task instance of a task when there is no task instance
# in the DB with an execution_date earlier than it., i.e. no manual marking
# success will be needed for a newly added task to be scheduled.
ignore_first_depends_on_past_by_default = True

# This changes the batch size of queries in the scheduling main loop.
# If this is too high, SQL query performance may be impacted by
# complexity of query predicate, and/or excessive locking.
# Additionally, you may hit the maximum allowable query length for your db.
# Set this to 0 for no limit (not advised)
max_tis_per_query = 512

# Should the scheduler issue ``SELECT ... FOR UPDATE`` in relevant queries.
# If this is set to False then you should not run more than a single
# scheduler at once
use_row_level_locking = True

# Max number of DAGs to create DagRuns for per scheduler loop.
max_dagruns_to_create_per_loop = 10

# How many DagRuns should a scheduler examine (and lock) when scheduling
# and queuing tasks.
max_dagruns_per_loop_to_schedule = 20

# Should the Task supervisor process perform a "mini scheduler" to attempt to schedule more tasks of the
# same DAG. Leaving this on will mean tasks in the same DAG execute quicker, but might starve out other
# dags in some circumstances
schedule_after_task_execution = True

# The scheduler can run multiple processes in parallel to parse dags.
# This defines how many processes will run.
parsing_processes = 2

# One of ``modified_time``, ``random_seeded_by_host`` and ``alphabetical``.
# The scheduler will list and sort the dag files to decide the parsing order.
#
# * ``modified_time``: Sort by modified time of the files. This is useful on large scale to parse the
#   recently modified DAGs first.
# * ``random_seeded_by_host``: Sort randomly across multiple Schedulers but with same order on the
#   same host. This is useful when running with Scheduler in HA mode where each scheduler can
#   parse different DAG files.
# * ``alphabetical``: Sort by filename
file_parsing_sort_mode = modified_time

# Whether the dag processor is running as a standalone process or it is a subprocess of a scheduler
# job.
standalone_dag_processor = False

# Only applicable if `[scheduler]standalone_dag_processor` is true and  callbacks are stored
# in database. Contains maximum number of callbacks that are fetched during a single loop.
max_callbacks_per_loop = 20

# Turn off scheduler use of cron intervals by setting this to False.
# DAGs submitted manually in the web UI or with trigger_dag will still run.
use_job_schedule = True

# Allow externally triggered DagRuns for Execution Dates in the future
# Only has effect if schedule_interval is set to None in DAG
allow_trigger_in_future = False

# DAG dependency detector class to use
dependency_detector = airflow.serialization.serialized_objects.DependencyDetector

# How often to check for expired trigger requests that have not run yet.
trigger_timeout_check_interval = 15

[triggerer]
# How many triggers a single Triggerer will run at once, by default.
default_capacity = 1000

[kerberos]
ccache = /tmp/airflow_krb5_ccache

# gets augmented with fqdn
principal = airflow
reinit_frequency = 3600
kinit_path = kinit
keytab = airflow.keytab

# Allow to disable ticket forwardability.
forwardable = True

# Allow to remove source IP from token, useful when using token behind NATted Docker host.
include_ip = True

[github_enterprise]
api_rev = v3

[elasticsearch]
# Elasticsearch host
host =

# Format of the log_id, which is used to query for a given tasks logs
log_id_template = {dag_id}-{task_id}-{run_id}-{map_index}-{try_number}

# Used to mark the end of a log stream for a task
end_of_log_mark = end_of_log

# Qualified URL for an elasticsearch frontend (like Kibana) with a template argument for log_id
# Code will construct log_id using the log_id template from the argument above.
# NOTE: scheme will default to https if one is not provided
# Example: frontend = http://localhost:5601/app/kibana#/discover?_a=(columns:!(message),query:(language:kuery,query:'log_id: "{log_id}"'),sort:!(log.offset,asc))
frontend =

# Write the task logs to the stdout of the worker, rather than the default files
write_stdout = False

# Instead of the default log formatter, write the log lines as JSON
json_format = False

# Log fields to also attach to the json output, if enabled
json_fields = asctime, filename, lineno, levelname, message

# The field where host name is stored (normally either `host` or `host.name`)
host_field = host

# The field where offset is stored (normally either `offset` or `log.offset`)
offset_field = offset

[elasticsearch_configs]
use_ssl = False
verify_certs = True

[kubernetes]
# Path to the YAML pod file that forms the basis for KubernetesExecutor workers.
pod_template_file = /home/airflow/pod_template_file.yaml

# The repository of the Kubernetes Image for the Worker to Run
worker_container_repository = apache/airflow

# The tag of the Kubernetes Image for the Worker to Run
worker_container_tag = latest-python3.10

# The Kubernetes namespace where airflow workers should be created. Defaults to ``default``
namespace = airflow

# If True, all worker pods will be deleted upon termination
delete_worker_pods = True

# If False (and delete_worker_pods is True),
# failed worker pods will not be deleted so users can investigate them.
# This only prevents removal of worker pods where the worker itself failed,
# not when the task it ran failed.
delete_worker_pods_on_failure = False

# Number of Kubernetes Worker Pod creation calls per scheduler loop.
# Note that the current default of "1" will only launch a single pod
# per-heartbeat. It is HIGHLY recommended that users increase this
# number to match the tolerance of their kubernetes cluster for
# better performance.
worker_pods_creation_batch_size = 8

# Allows users to launch pods in multiple namespaces.
# Will require creating a cluster-role for the scheduler
multi_namespace_mode = False

# Use the service account kubernetes gives to pods to connect to kubernetes cluster.
# It's intended for clients that expect to be running inside a pod running on kubernetes.
# It will raise an exception if called from a process not running in a kubernetes environment.
in_cluster = False

# When running with in_cluster=False change the default cluster_context or config_file
# options to Kubernetes client. Leave blank these to use default behaviour like ``kubectl`` has.
cluster_context = microk8s

# Path to the kubernetes configfile to be used when ``in_cluster`` is set to False
config_file = /home/airflow/kube-config

# Keyword parameters to pass while calling a kubernetes client core_v1_api methods
# from Kubernetes Executor provided as a single line formatted JSON dictionary string.
# List of supported params are similar for all core_v1_apis, hence a single config
# variable for all apis. See:
# https://raw.githubusercontent.com/kubernetes-client/python/41f11a09995efcd0142e25946adc7591431bfb2f/kubernetes/client/api/core_v1_api.py
kube_client_request_args =

# Optional keyword arguments to pass to the ``delete_namespaced_pod`` kubernetes client
# ``core_v1_api`` method when using the Kubernetes Executor.
# This should be an object and can contain any of the options listed in the ``v1DeleteOptions``
# class defined here:
# https://github.com/kubernetes-client/python/blob/41f11a09995efcd0142e25946adc7591431bfb2f/kubernetes/client/models/v1_delete_options.py#L19
# Example: delete_option_kwargs = {"grace_period_seconds": 10}
delete_option_kwargs =

# Enables TCP keepalive mechanism. This prevents Kubernetes API requests to hang indefinitely
# when idle connection is time-outed on services like cloud load balancers or firewalls.
enable_tcp_keepalive = True

# When the `enable_tcp_keepalive` option is enabled, TCP probes a connection that has
# been idle for `tcp_keep_idle` seconds.
tcp_keep_idle = 120

# When the `enable_tcp_keepalive` option is enabled, if Kubernetes API does not respond
# to a keepalive probe, TCP retransmits the probe after `tcp_keep_intvl` seconds.
tcp_keep_intvl = 30

# When the `enable_tcp_keepalive` option is enabled, if Kubernetes API does not respond
# to a keepalive probe, TCP retransmits the probe `tcp_keep_cnt number` of times before
# a connection is considered to be broken.
tcp_keep_cnt = 6

# Set this to false to skip verifying SSL certificate of Kubernetes python client.
verify_ssl = True

# How long in seconds a worker can be in Pending before it is considered a failure
worker_pods_pending_timeout = 300

# How often in seconds to check if Pending workers have exceeded their timeouts
worker_pods_pending_timeout_check_interval = 120

# How often in seconds to check for task instances stuck in "queued" status without a pod
worker_pods_queued_check_interval = 60

# How many pending pods to check for timeout violations in each check interval.
# You may want this higher if you have a very large cluster and/or use ``multi_namespace_mode``.
worker_pods_pending_timeout_batch_size = 100

[sensors]
# Sensor default timeout, 7 days by default (7 * 24 * 60 * 60).
default_timeout = 604800

[smart_sensor]
# When `use_smart_sensor` is True, Airflow redirects multiple qualified sensor tasks to
# smart sensor task.
use_smart_sensor = False

# `shard_code_upper_limit` is the upper limit of `shard_code` value. The `shard_code` is generated
# by `hashcode % shard_code_upper_limit`.
shard_code_upper_limit = 10000

# The number of running smart sensor processes for each service.
shards = 5

# comma separated sensor classes support in smart_sensor.
sensors_enabled = NamedHivePartitionSensor
```

**NOTE: Some of these configs require us to change some settings in the Airflow UI later.**

Create new folders inside the "airflow/" dir for our DAGs to functions correctly:

```
airflow/dags
airflow/plugins
airflow/temp
```

Next up, initialise Backend DB for Airflow:

```
airflow db init
```

If you've done everything correctly, you should see ```Initialization done```.

Next up, create a user with Admin role to access our Airflow Web App:
```
# create an admin user
airflow users create \
    --username admin \
    --password admin \
    --firstname Admin \
    --lastname Admin \
    --role Admin \
    --email dungnk@ikame.vn
```

Next, head to ```https://airflow.bi.ikameglobal.com/connection/list/```

Select all of the connections -> Actions -> Delete.

Now create a file called ```airflow-connections.json``` in the HOME dir of the user "airflow", with the following content:
```
{
  "airflow_db": {
    "conn_type": "mysql",
    "description": null,
    "login": "root",
    "password": null,
    "host": "mysql",
    "port": null,
    "schema": "airflow",
    "extra": null
  },
  "aws_default": {
    "conn_type": "aws",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": null
  },
  "azure_batch_default": {
    "conn_type": "azure_batch",
    "description": null,
    "login": "<ACCOUNT_NAME>",
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\"account_url\": \"<ACCOUNT_URL>\"}"
  },
  "azure_cosmos_default": {
    "conn_type": "azure_cosmos",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\"database_name\": \"<DATABASE_NAME>\", \"collection_name\": \"<COLLECTION_NAME>\" }"
  },
  "azure_data_explorer_default": {
    "conn_type": "azure_data_explorer",
    "description": null,
    "login": null,
    "password": null,
    "host": "https://<CLUSTER>.kusto.windows.net",
    "port": null,
    "schema": null,
    "extra": "{\"auth_method\": \"<AAD_APP | AAD_APP_CERT | AAD_CREDS | AAD_DEVICE>\",\n                    \"tenant\": \"<TENANT ID>\", \"certificate\": \"<APPLICATION PEM CERTIFICATE>\",\n                    \"thumbprint\": \"<APPLICATION CERTIFICATE THUMBPRINT>\"}"
  },
  "azure_data_lake_default": {
    "conn_type": "azure_data_lake",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\"tenant\": \"<TENANT>\", \"account_name\": \"<ACCOUNTNAME>\" }"
  },
  "azure_default": {
    "conn_type": "azure",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": null
  },
  "cassandra_default": {
    "conn_type": "cassandra",
    "description": null,
    "login": null,
    "password": null,
    "host": "cassandra",
    "port": 9042,
    "schema": null,
    "extra": null
  },
  "databricks_default": {
    "conn_type": "databricks",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": null,
    "schema": null,
    "extra": null
  },
  "dingding_default": {
    "conn_type": "http",
    "description": null,
    "login": null,
    "password": null,
    "host": "",
    "port": null,
    "schema": null,
    "extra": null
  },
  "drill_default": {
    "conn_type": "drill",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 8047,
    "schema": null,
    "extra": "{\"dialect_driver\": \"drill+sadrill\", \"storage_plugin\": \"dfs\"}"
  },
  "druid_broker_default": {
    "conn_type": "druid",
    "description": null,
    "login": null,
    "password": null,
    "host": "druid-broker",
    "port": 8082,
    "schema": null,
    "extra": "{\"endpoint\": \"druid/v2/sql\"}"
  },
  "druid_ingest_default": {
    "conn_type": "druid",
    "description": null,
    "login": null,
    "password": null,
    "host": "druid-overlord",
    "port": 8081,
    "schema": null,
    "extra": "{\"endpoint\": \"druid/indexer/v1/task\"}"
  },
  "elasticsearch_default": {
    "conn_type": "elasticsearch",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 9200,
    "schema": "http",
    "extra": null
  },
  "emr_default": {
    "conn_type": "emr",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "\n                {   \"Name\": \"default_job_flow_name\",\n                    \"LogUri\": \"s3://my-emr-log-bucket/default_job_flow_location\",\n                    \"ReleaseLabel\": \"emr-4.6.0\",\n                    \"Instances\": {\n                        \"Ec2KeyName\": \"mykey\",\n                        \"Ec2SubnetId\": \"somesubnet\",\n                        \"InstanceGroups\": [\n                            {\n                                \"Name\": \"Master nodes\",\n                                \"Market\": \"ON_DEMAND\",\n                                \"InstanceRole\": \"MASTER\",\n                                \"InstanceType\": \"r3.2xlarge\",\n                                \"InstanceCount\": 1\n                            },\n                            {\n                                \"Name\": \"Core nodes\",\n                                \"Market\": \"ON_DEMAND\",\n                                \"InstanceRole\": \"CORE\",\n                                \"InstanceType\": \"r3.2xlarge\",\n                                \"InstanceCount\": 1\n                            }\n                        ],\n                        \"TerminationProtected\": false,\n                        \"KeepJobFlowAliveWhenNoSteps\": false\n                    },\n                    \"Applications\":[\n                        { \"Name\": \"Spark\" }\n                    ],\n                    \"VisibleToAllUsers\": true,\n                    \"JobFlowRole\": \"EMR_EC2_DefaultRole\",\n                    \"ServiceRole\": \"EMR_DefaultRole\",\n                    \"Tags\": [\n                        {\n                            \"Key\": \"app\",\n                            \"Value\": \"analytics\"\n                        },\n                        {\n                            \"Key\": \"environment\",\n                            \"Value\": \"development\"\n                        }\n                    ]\n                }\n            "
  },
  "facebook_default": {
    "conn_type": "facebook_social",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "\n                {   \"account_id\": \"<AD_ACCOUNT_ID>\",\n                    \"app_id\": \"<FACEBOOK_APP_ID>\",\n                    \"app_secret\": \"<FACEBOOK_APP_SECRET>\",\n                    \"access_token\": \"<FACEBOOK_AD_ACCESS_TOKEN>\"\n                }\n            "
  },
  "fs_default": {
    "conn_type": "fs",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\"path\": \"/\"}"
  },
  "google_cloud_default": {
    "conn_type": "google_cloud_platform",
    "description": "",
    "login": "admin",
    "password": "admin",
    "host": "",
    "port": null,
    "schema": "default",
    "extra": "{\"extra__google_cloud_platform__key_path\": \"/home/airflow/bi-team-admin-secrets.json\", \"extra__google_cloud_platform__num_retries\": 5, \"extra__google_cloud_platform__project\": \"zegobi-datacenters\"}"
  },
  "hive_cli_default": {
    "conn_type": "hive_cli",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 10000,
    "schema": "default",
    "extra": "{\"use_beeline\": true, \"auth\": \"\"}"
  },
  "hiveserver2_default": {
    "conn_type": "hiveserver2",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 10000,
    "schema": "default",
    "extra": null
  },
  "http_default": {
    "conn_type": "http",
    "description": null,
    "login": null,
    "password": null,
    "host": "https://www.httpbin.org/",
    "port": null,
    "schema": null,
    "extra": null
  },
  "kubernetes_default": {
    "conn_type": "kubernetes",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": null
  },
  "kylin_default": {
    "conn_type": "kylin",
    "description": null,
    "login": "ADMIN",
    "password": "KYLIN",
    "host": "localhost",
    "port": 7070,
    "schema": null,
    "extra": null
  },
  "leveldb_default": {
    "conn_type": "leveldb",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": null,
    "schema": null,
    "extra": null
  },
  "livy_default": {
    "conn_type": "livy",
    "description": null,
    "login": null,
    "password": null,
    "host": "livy",
    "port": 8998,
    "schema": null,
    "extra": null
  },
  "local_mysql": {
    "conn_type": "mysql",
    "description": null,
    "login": "airflow",
    "password": "airflow",
    "host": "localhost",
    "port": null,
    "schema": "airflow",
    "extra": null
  },
  "metastore_default": {
    "conn_type": "hive_metastore",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 9083,
    "schema": null,
    "extra": "{\"authMechanism\": \"PLAIN\"}"
  },
  "mongo_default": {
    "conn_type": "mongo",
    "description": null,
    "login": null,
    "password": null,
    "host": "mongo",
    "port": 27017,
    "schema": null,
    "extra": null
  },
  "mssql_default": {
    "conn_type": "mssql",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 1433,
    "schema": null,
    "extra": null
  },
  "mysql_default": {
    "conn_type": "mysql",
    "description": null,
    "login": "root",
    "password": null,
    "host": "mysql",
    "port": null,
    "schema": "airflow",
    "extra": null
  },
  "opsgenie_default": {
    "conn_type": "http",
    "description": null,
    "login": null,
    "password": null,
    "host": "",
    "port": null,
    "schema": null,
    "extra": null
  },
  "oss_default": {
    "conn_type": "oss",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\n                \"auth_type\": \"AK\",\n                \"access_key_id\": \"<ACCESS_KEY_ID>\",\n                \"access_key_secret\": \"<ACCESS_KEY_SECRET>\",\n                \"region\": \"<YOUR_OSS_REGION>\"}\n                "
  },
  "pig_cli_default": {
    "conn_type": "pig_cli",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": "default",
    "extra": null
  },
  "pinot_admin_default": {
    "conn_type": "pinot",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 9000,
    "schema": null,
    "extra": null
  },
  "pinot_broker_default": {
    "conn_type": "pinot",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 9000,
    "schema": null,
    "extra": "{\"endpoint\": \"/query\", \"schema\": \"http\"}"
  },
  "postgres_default": {
    "conn_type": "postgres",
    "description": null,
    "login": "postgres",
    "password": "airflow",
    "host": "postgres",
    "port": null,
    "schema": "airflow",
    "extra": null
  },
  "presto_default": {
    "conn_type": "presto",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 3400,
    "schema": "hive",
    "extra": null
  },
  "qubole_default": {
    "conn_type": "qubole",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": null,
    "schema": null,
    "extra": null
  },
  "redis_default": {
    "conn_type": "redis",
    "description": null,
    "login": null,
    "password": null,
    "host": "redis",
    "port": 6379,
    "schema": null,
    "extra": "{\"db\": 0}"
  },
  "redshift_default": {
    "conn_type": "redshift",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\n    \"iam\": true,\n    \"cluster_identifier\": \"<REDSHIFT_CLUSTER_IDENTIFIER>\",\n    \"port\": 5439,\n    \"profile\": \"default\",\n    \"db_user\": \"awsuser\",\n    \"database\": \"dev\",\n    \"region\": \"\"\n}"
  },
  "segment_default": {
    "conn_type": "segment",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\"write_key\": \"my-segment-write-key\"}"
  },
  "sendgrid_default": {
    "conn_type": "email",
    "description": "",
    "login": "apikey",
    "password": "*********************",
    "host": "smtp.sendgrid.net",
    "port": 587,
    "schema": "",
    "extra": ""
  },
  "sftp_default": {
    "conn_type": "sftp",
    "description": null,
    "login": "airflow",
    "password": null,
    "host": "localhost",
    "port": 22,
    "schema": null,
    "extra": "{\"key_file\": \"~/.ssh/id_rsa\", \"no_host_key_check\": true}"
  },
  "spark_default": {
    "conn_type": "spark",
    "description": null,
    "login": null,
    "password": null,
    "host": "yarn",
    "port": null,
    "schema": null,
    "extra": "{\"queue\": \"root.default\"}"
  },
  "sqlite_default": {
    "conn_type": "sqlite",
    "description": null,
    "login": null,
    "password": null,
    "host": "/tmp/sqlite_default.db",
    "port": null,
    "schema": null,
    "extra": null
  },
  "sqoop_default": {
    "conn_type": "sqoop",
    "description": null,
    "login": null,
    "password": null,
    "host": "rdbms",
    "port": null,
    "schema": null,
    "extra": null
  },
  "ssh_default": {
    "conn_type": "ssh",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": null,
    "schema": null,
    "extra": null
  },
  "tableau_default": {
    "conn_type": "tableau",
    "description": null,
    "login": "user",
    "password": "password",
    "host": "https://tableau.server.url",
    "port": null,
    "schema": null,
    "extra": "{\"site_id\": \"my_site\"}"
  },
  "trino_default": {
    "conn_type": "trino",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 3400,
    "schema": "hive",
    "extra": null
  },
  "vertica_default": {
    "conn_type": "vertica",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 5433,
    "schema": null,
    "extra": null
  },
  "wasb_default": {
    "conn_type": "wasb",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": null,
    "extra": "{\"sas_token\": null}"
  },
  "webhdfs_default": {
    "conn_type": "hdfs",
    "description": null,
    "login": null,
    "password": null,
    "host": "localhost",
    "port": 50070,
    "schema": null,
    "extra": null
  },
  "yandexcloud_default": {
    "conn_type": "yandexcloud",
    "description": null,
    "login": null,
    "password": null,
    "host": null,
    "port": null,
    "schema": "default",
    "extra": null
  }
}
```
*You need to get your own SendGrid API Key to replace the "**********" part.*

Run:
```
airflow connections import airflow-connections.json
```

You should see the following output:
```
Imported connection airflow_db
Imported connection aws_default
Imported connection azure_batch_default
Imported connection azure_cosmos_default
Imported connection azure_data_explorer_default
Imported connection azure_data_lake_default
Imported connection azure_default
Imported connection cassandra_default
Imported connection databricks_default
Imported connection dingding_default
Imported connection drill_default
Imported connection druid_broker_default
Imported connection druid_ingest_default
Imported connection elasticsearch_default
Imported connection emr_default
Imported connection facebook_default
Imported connection fs_default
Imported connection google_cloud_default
Imported connection hive_cli_default
Imported connection hiveserver2_default
Imported connection http_default
Imported connection kubernetes_default
Imported connection kylin_default
Imported connection leveldb_default
Imported connection livy_default
Imported connection local_mysql
Imported connection metastore_default
Imported connection mongo_default
Imported connection mssql_default
Imported connection mysql_default
Imported connection opsgenie_default
Imported connection oss_default
Imported connection pig_cli_default
Imported connection pinot_admin_default
Imported connection pinot_broker_default
Imported connection postgres_default
Imported connection presto_default
Imported connection qubole_default
Imported connection redis_default
Imported connection redshift_default
Imported connection segment_default
Imported connection sendgrid_default
Imported connection sftp_default
Imported connection spark_default
Imported connection sqlite_default
Imported connection sqoop_default
Imported connection ssh_default
Imported connection tableau_default
Imported connection trino_default
Imported connection vertica_default
Imported connection wasb_default
Imported connection webhdfs_default
Imported connection yandexcloud_default
```

We are done here for Airflow.

## Set up KubernetesExecutor

Create a pod_template_file.yaml with the following contents.

**Follow instructions inside the .yaml file.**

```
apiVersion: v1
kind: Pod
metadata:
  # Name is not important here,
  # as it will be overidden by airflow/k8s.
  # LEAVE DEFAULT.
  name: dummy-name

  # RUN: kubectl create namespace airflow
  # This is where our worker pods will be created in.
  # LEAVE DEFAULT.
  namespace: airflow

spec:
  containers:
    - env:
        # Airflow HOME
        # This is just for our k8s pods,
        # therefore it has a different HOME path from
        # our local Airflow deployment.
        # LEAVE DEFAULT.
        - name: AIRFLOW_HOME
          value: /opt/airflow/

        # Our k8s pods will read DAGs from this
        # folder inside them,
        # and this folder is linked to our host's local storage.
        # So essentially, our pods are reading DAGs directly
        # from our host's local storage.
        # LEAVE DEFAULT.
        - name: AIRFLOW__CORE__DAGS_FOLDER
          value: /opt/airflow/dags

        # Executor
        # LEAVE DEFAULT.
        - name: AIRFLOW__CORE__EXECUTOR
          value: LocalExecutor

        # LEAVE DEFAULT.
        - name: AIRFLOW__CORE__ENABLE_XCOM_PICKLING
          value: true

        # Database
        # Read secrets from our secretKeyRef, which is stored
        # by our microk8s cluster.
        # LEAVE DEFAULT.
        - name: AIRFLOW__CORE__FERNET_KEY
          valueFrom:
            secretKeyRef:
              name: airflow-k8s-pods
              key: fernet-key

        # LEAVE DEFAULT.
        - name: AIRFLOW__DATABASE__SQL_ALCHEMY_CONN
          valueFrom:
            secretKeyRef:
              name: airflow-k8s-pods
              key: postgresql_connection

        # LEAVE DEFAULT.
        - name: AIRFLOW_CONN_AIRFLOW_DB
          valueFrom:
            secretKeyRef:
              name: airflow-k8s-pods
              key: postgresql_connection

        # Remote logging to GCS
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /opt/airflow/bi-team-admin-secrets.json

        - name: AIRFLOW__LOGGING__REMOTE_LOGGING
          value: "True"

        - name: AIRFLOW__LOGGING__REMOTE_LOG_CONN_ID
          value: google_cloud_default

        - name: AIRFLOW__API__GOOGLE_KEY_PATH
          value: /opt/airflow/bi-team-admin-secrets.json

        - name: AIRFLOW__LOGGING__REMOTE_BASE_LOG_FOLDER
          value: gs://airflow-vps-logging-bi-team/logs

        # Send email notifications via SendGrid
        - name: AIRFLOW__EMAIL__EMAIL_BACKEND
          value: airflow.providers.sendgrid.utils.emailer.send_email

        - name: AIRFLOW__EMAIL__EMAIL_CONN_ID
          value: sendgrid_default

        - name: SENDGRID_MAIL_FROM
          value: botNKD <dungnk@ikame.vn>

        - name: SENDGRID_API_KEY
          valueFrom:
            secretKeyRef:
              name: airflow-k8s-pods
              key: sendgrid_apikey

      # Image is not important here,
      # as it will be overidden by airflow/k8s.
      # LEAVE DEFAULT.
      image: dummy_image
      imagePullPolicy: IfNotPresent
      # MUST HAVE A CONTAINER NAMED "base".
      # LEAVE DEFAULT.
      name: base
      # Mount ephemeral storages inside pods for Airflow to work
      # correctly. These are linked to the "volumes" section below.
      # "name" of volumeMounts and volumes must match for the directories
      # to be linked.
      # LEAVE DEFAULT.
      volumeMounts:
        - name: airflow-logs
          readOnly: false
          mountPath: /opt/airflow/logs

        - name: airflow-dags
          readOnly: true
          mountPath: /opt/airflow/dags

        - name: airflow-bi-team-secrets
          readOnly: true
          mountPath: /opt/airflow/bi-team-admin-secrets.json
          subPath: bi-team-admin-secrets.json

        - name: airflow-begamob-secrets
          readOnly: true
          mountPath: /opt/airflow/begamob_client_secrets.json
          subPath: begamob_client_secrets.json

        - name: airflow-requirements
          readOnly: true
          mountPath: /opt/airflow/requirements.txt
          subPath: requirements.txt

  restartPolicy: Never
  # MUST RUN PODS AS USER "airflow". Check what ID is user "airflow"
  # and what ID is its group by running:
  # userID: id -u airflow
  # groupID: id -g airflow
  # Replace "runAsUser" with the value of userID above.
  # Replace "fsGroup" with the value of groupID above.
  securityContext:
    runAsUser: 1001
    fsGroup: 1002
  # LEAVE DEFAULT.
  serviceAccountName: "default"
  # These paths are from our host's local storage. We are essentially
  # mounting our host's local storage onto our pods.
  # If you are running as user "airflow" as you should be, then leave
  # these configs alone.
  volumes:
    - name: airflow-dags
      hostPath:
        path: /home/airflow/airflow/dags

    - name: airflow-logs
      emptyDir: {}

    - name: airflow-bi-team-secrets
      hostPath:
        path: /home/airflow

    - name: airflow-begamob-secrets
      hostPath:
        path: /home/airflow

    - name: airflow-requirements
      hostPath:
        path: /home/airflow
```

Then, create a kube-config file for our local Airflow deployment to talk to the K8s cluster, by running:

```
microk8s config >> kube-config
```

In order for our DAGs to authenticate to Google services, create these files inside the HOME folder of the user "airflow":
```
bi-team-admin-secrets.json
begamob_client_secrets.json
```

> Contact skype: kdunggttn1 (Nguyễn Khắc Dũng - Data Engineer) to get these files.

## Set up K8s cluster

Since we will be running KubernetesExecutor, we will need to access the K8s Dashboard for easy monitoring and managing pods and namespaces.

First, set k8s to use nano as its default editor. Unless you want something harder of course.
```
which nano
```

```
Output
> /usr/bin/nano
```

```
sudo nano ~/.bashrc
```

Add this line to the end:
```
# Config KUBE_EDITOR
export KUBE_EDITOR="/usr/bin/nano"
```

Then do: 
```
source ~/.bashrc
``` 
to save changes.

Create a directory in the HOME path called ```/certs```. Do:
```
cd certs
```
then run these commands:
```
openssl genrsa -des3 -passout pass:over4chars -out dashboard.pass.key 2048
...
openssl rsa -passin pass:over4chars -in dashboard.pass.key -out dashboard.key
# Writing RSA key
rm dashboard.pass.key
openssl req -new -key dashboard.key -out dashboard.csr
...
Country Name (2 letter code) [AU]: US
...
A challenge password []:
...
```

Then generate SSL cert with:
```
openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt
```

Custom certificates have to be stored in a secret named ```kubernetes-dashboard-certs``` in the same namespace as Kubernetes Dashboard. Assuming that you have ```*.crt``` and ```*.key``` files stored under ```$HOME/certs``` directory, you should create secret with contents of these files:

```
kubectl delete secrets/kubernetes-dashboard-certs -n kube-system && kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kube-system
```

Then, edit the deployment for "kubenetes-dashboard" by running:
```
kubectl edit deployments/kubernetes-dashboard -n kube-system
```

After these args,
```
- --auto-generate-certificates
- --namespace=kube-system
```
add:
```
- --tls-cert-file=/airflow/certs/dashboard.crt
- --tls-key-file=/airflow/certs/dashboard.key
- --enable-skip-login
- --disable-settings-authorizer
```
So the final result would look like this:
```
spec:
  containers:
  - args:
    - --auto-generate-certificates
    - --namespace=kube-system
    - --tls-cert-file=/root/certs/dashboard.crt
    - --tls-key-file=/airflow/certs/dashboard.key
    - --enable-skip-login
    - --disable-settings-authorizer
    image: kubernetesui/dashboard:v2.3.0
    imagePullPolicy: IfNotPresent
```
**NOTE: YOU MUST USE "SPACE" FOR INDENTATION AND NOT "TAB".**

Save and exit by ```Ctrl + X```.

Create a systemd service to manage K8s dashboard by doing:
```
sudo nano /etc/systemd/system/k8s-dashboard.service
```
```
[Unit]
Description=K8s - Dashboard
After=network.target

[Service]
User=root
ExecStart=/snap/bin/microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

The "ExecStart" command makes our k8s port-forwards the service "kubernetes-dashboard" in the namespace "kube-system" running on port 10443 to port 443, which will be handled by our Nginx.

Run the K8s Dashboard:
```
sudo systemctl start k8s-dashboard
sudo systemctl enable k8s-dashboard
```

Now that we have K8s dashboard up and running, head to ```https://airflow.bi.ikameglobal.com/k8s/#/secret?namespace=airflow```

Create a new secret by clicking on the "+" icon on the top right corner.

Choose "Create from input" and paste in these:
```
apiVersion: v1
data:
  fernet-key: **************
  postgresql_connection: **************
  sendgrid_apikey: **************
kind: Secret
metadata:
  name: airflow-k8s-pods
  namespace: airflow
type: Opaque
```

> fernet-key: Copy the fernet-key from ```airflow.cfg``` and paste here.

> postgresql_connection: The psql connection string in ```airflow.cfg```.

> sendgrid_apikey: Again, get your own SendGrid API key.

***Remember that all these keys must be base64-encoded for K8s to read them correctly.***

And that's it for our K8s Dashboard.

## Set up CeleryExecutor

We don't need to set up anything for CeleryExecutor here.



# IV. Set up Nginx as reverse-proxy for our Airflow services

So far, we have successfully deployed various Airflow services, namely Webserver, Scheduler, Celery Flower, Celery Worker and K8s Dashboard. In order to access these services from outside the VPS (and preferably with a domain), we need Nginx for all our routing, renaming and cert-ing needs.

Start by installing nginx:
```
sudo apt install nginx -Y
```

Next, create a configuration file for our domain "airflow.bi.ikameglobal.com":
```
sudo nano /etc/nginx/sites-available/airflow.bi.ikameglobal.com
```
```
server {

    listen 80 default_server;
	listen [::]:80 default_server;

	server_name www.airflow.bi.ikameglobal.com airflow.bi.ikameglobal.com;

	location / {
        proxy_pass http://localhost:1234;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
	}

	location /k8s {
        proxy_pass https://localhost:10443;
        rewrite \/k8s\/(.*) /$1 break;
        # proxy_http_version 1.1;
        # proxy_set_header Upgrade $http_upgrade;
        # proxy_set_header Connection 'upgrade';
        # proxy_set_header Host $host;
        # proxy_cache_bypass $http_upgrade;
    }

	location /flower {
        proxy_pass http://localhost:5555;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
> Explain: "rewrite \/k8s\/(.*) /$1 break;" under "location /k8s" means we will be able to access our k8s dashboard by typing in "airflow.bi.ikameglobal.com/k8s" and nginx will automatically rewrite the URL for us, so k8s dashboard will NOT return 404 page not found (since k8s dashboard only works with "/", meaning it only receives requests from "airflow.bi.ikameglobal.com/").

> Since we defined the URL suffix for our Flower Dashboard in airflow.cfg, we only need to point nginx to "/flower" here, and no further URL rewrite is needed.

Save and exit.

Delete the symlink for the default config created by nginx, to avoid conflicts with our own:
```
sudo rm /etc/nginx/sites-enabled/default
```

Next, create a symlink for our config:
```
sudo ln -s /etc/nginx/sites-available/airflow.bi.ikameglobal.com /etc/nginx/sites-enabled/airflow.bi.ikameglobal.com
```

Restart nginx for changes to take effect:
```
sudo systemctl restart nginx
```

***Next, ask the SysAdmin/DevOps to point the subdomain www.airflow.bi and airflow.bi to out VPS' IP Address. In order to successfully obtain SSL certs, use "DNS only" instead of "proxied".***

> Contact skype: songvidamme (Nguyễn Đức Long - Team Backend) for all your domain needs.

**IMPORTANT!**: Obtain SSL certs for our domains. We are going to use CertBot for this.

Ensure that the subdomains have been pointed to our VPS.

Install CertBot:
```
pip install certbot certbot-nginx
sudo apt install certbot python3-certbot-nginx
```

Allowing HTTPS Through the Firewall:
```
sudo ufw allow 'Nginx Full'
sudo ufw delete allow 'Nginx HTTP'
```

Check firewall status:
```
sudo ufw status
```

Next, obtain SSL certs for our subdomains by running:
```
sudo certbot
```

You should get similar output:
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): dungnk@ikame.vn

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel: A

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: N

Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: airflow.bi.ikameglobal.com
2: www.airflow.bi.ikameglobal.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): 
```

Press Enter to get certs for all of our subdomains defined in our custom nginx config file.

If you've done things correctly, certbot should return:
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx

Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: airflow.bi.ikameglobal.com
2: www.airflow.bi.ikameglobal.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): 
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for airflow.bi.ikameglobal.com
http-01 challenge for www.airflow.bi.ikameglobal.com
Waiting for verification...
Cleaning up challenges
Deploying Certificate to VirtualHost /etc/nginx/sites-enabled/airflow.bi.ikameglobal.com
Deploying Certificate to VirtualHost /etc/nginx/sites-enabled/airflow.bi.ikameglobal.com

Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
Redirecting all traffic on port 80 to ssl in /etc/nginx/sites-enabled/airflow.bi.ikameglobal.com
Redirecting all traffic on port 80 to ssl in /etc/nginx/sites-enabled/airflow.bi.ikameglobal.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations! You have successfully enabled
https://airflow.bi.ikameglobal.com and https://www.airflow.bi.ikameglobal.com

You should test your configuration at:
https://www.ssllabs.com/ssltest/analyze.html?d=airflow.bi.ikameglobal.com
https://www.ssllabs.com/ssltest/analyze.html?d=www.airflow.bi.ikameglobal.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/airflow.bi.ikameglobal.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/airflow.bi.ikameglobal.com/privkey.pem
   Your cert will expire on 2022-10-13. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

# V. Git-clone our DAGs to /airflow/dags (OPTIONAL)

We are going to use a GitHub repo to store our DAGs (so we can code directly on our VPS & commit changes instantly).

First we need to store our Git credentials for peace of mind:
```
git config --global credential.helper store
```

Config git user name:
```
git config --global user.name "NKD"
```

Config git user email:
```
git config --global user.email "dungnk@ikame.vn"
```

Next, do a ```git clone```:
```
git clone https://github.com/kdunggttn/ikame_airflow_dags
```

# V. Test our DAGs as well as our Airflow deployment

Create a DAG with the following content:

```
import datetime

from airflow import models
from airflow.decorators import task

dag_name = 'test_DAG'

@task
def _get_dates(**kwargs):

    import pandas as pd

    # Get today's date
    now = datetime.datetime.now()
    today = now.strftime('%Y-%m-%d')
    last_3_days = now - datetime.timedelta(days=3)
    last_3_days = last_3_days.strftime('%Y-%m-%d')

    start_date = last_3_days
    end_date = today

    daterange:list = pd.date_range(start=start_date, end=end_date).strftime('%Y-%m-%d').tolist()

    return daterange

@task
def _dynamic_task_mapping():

    task_list = []
    for i in range(10):
        task_list.append(i)
    
    return task_list

@task
def _enumerate_CeleryExecutor(enumeration, date):

    print(f'This is enumeration {enumeration}.')
    print(f'And the date is {date}.')

@task(
    queue='kubernetes'
)
def _enumerate_KubernetesExecutor(enumeration, date):

    print(f'This is enumeration {enumeration}.')
    print(f'And the date is {date}.')

default_dag_args = {
    'owner': 'dungnk@ikame.vn',
    # The start_date describes when a DAG is valid / can be run. Set this to a
    # fixed point in time rather than dynamically, since it is evaluated every
    # time a DAG is parsed. See:
    # https://airflow.apache.org/faq.html#what-s-the-deal-with-start-date
    'start_date': datetime.datetime(2022, 1, 1),
    # Email whenever an Operator in the DAG fails.
    'email': ['dungnk@ikame.vn', 'bi.connector@ikame.vn'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 5,
    'retry_delay': datetime.timedelta(minutes=10),
}

# Define a DAG (directed acyclic graph) of tasks.
with models.DAG(
        dag_name,
        description='Test DAG with both CeleryExecutor and KubernetesExecutor',
        schedule_interval=None,
        default_args=default_dag_args,
        catchup=False,
    ) as dag:


    enumerate_CeleryExecutor = _enumerate_CeleryExecutor.expand(
        enumeration=_dynamic_task_mapping(),
        date=_get_dates()
    )

    enumerate_KubernetesExecutor = _enumerate_KubernetesExecutor.expand(
        enumeration=_dynamic_task_mapping(),
        date=_get_dates()
    )

    # Define the order in which the tasks complete by using the >> and <<
    # operators.
    [enumerate_CeleryExecutor, enumerate_KubernetesExecutor]
```

If everything is set up correctly, the DAG should run flawlessly, with ```enumerate_CeleryExecutor``` tasks executed by Celery, and ```enumerate_KubernetesExecutor``` tasks executed by our K8s cluster. To check if the pods and their corresponding logs are working as intended, go to K8s dashboard -> namespace/airflow -> pods.

# VI. (OPTIONAL) Config timezone, hostname... for our VPS

In order for Airflow, as well as our DAGs, to read time correctly, we need to set up the VPS timezone.
```
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
```

To check if the timezone has been set correctly, run:
```
timedatectl
```
```
               Local time: Sat 2022-07-16 03:59:13 +07  
           Universal time: Fri 2022-07-15 20:59:13 UTC  
                 RTC time: Fri 2022-07-15 20:59:13      
                Time zone: Asia/Ho_Chi_Minh (+07, +0700)
System clock synchronized: yes                          
              NTP service: active                       
          RTC in local TZ: no
```

Do a reboot for the time setting to take effect
```
sudo reboot
```

Now all the timestamps in our DAGs' tasks will be correct.

---
# CONCLUSION

There you have it. A full-featured Airflow deployment running on a VPS, getting the best of both worlds, Celery and Kubernetes, without having to subject to limitations inherent to KEDA (Kubernetes Event-Driven Autoscaling), where every pod gets the same specs.

Should you have any questions, contact me via:
> Skype: kdunggttn1

> Email: dungnk@ikame.vn
