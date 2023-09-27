---
title: Troubleshooting and debugging your PostHog instance
sidebar: Docs
showTitle: true
---

import Sunset from "../_snippets/sunset-disclaimer.mdx"

<Sunset />

## Troubleshooting

### Connection is not secure

First, check that DNS is set up properly:

```shell
nslookup <your-hostname> 1.1.1.1
```

Note that when using a browser there are various layers of caching and other logic that could make the resolution work (temporarily) even if its not correctly set up.

## FAQ

### Are the errors I'm seeing important?

Here are some examples of log spam that currently exists in our app and is safe to ignore:

The following messages in the ClickHouse pod happen when ClickHouse reshuffles how it consumes from the topics. So, anytime ClickHouse or Kafka restarts we'll get a bit of noise and the following log entries are safe to ignore:

```
<Error> TCPHandler: Code: 60, e.displayText() = DB::Exception: Table posthog.sharded_events doesn't exist.
...
<Warning> StorageKafka (kafka_session_recording_events): Can't get assignment. It can be caused by some issue with consumer group (not enough partitions?). Will keep trying.
```

The following error is produced by some low-priority celery tasks and we haven't seen any actual impact so can safely be ignored. It shows up in Sentry as well.

```
TooManyConnections: too many connections
  File "posthog/celery.py",
  ...
  File "clickhouse_pool/pool.py", line 102, in pull
    raise TooManyConnections("too many connections")
```

### How do I see logs for a pod?

1. Find the name of the pod you want to get logs on:

    ```shell
    kubectl get pods -n posthog
    ```

    This command will list all running pods. If you want app/plugin server logs, for example, look for a pod that has a name starting with `posthog-plugins`. This will be something like `posthog-plugins-54f324b649-66afm`

2. Get the logs for that pod using the name from the previous step:

    ```bash
    kubectl logs posthog-plugins-54f324b649-66afm -n posthog
    ```

### How do I connect to the web server's shell?

PostHog is built on Django, which comes with some useful utilities. One of them is a Python shell.
You can connect to it like so:

```bash
# First we need to determine the name of the web pod – see "How do I see logs for a pod?" for more on this
POSTHOG_WEB_POD_NAME=$(kubectl get pods -n posthog | grep -- '-web-' | awk '{print $1}')
# Then we connect to the interactive Django shell
kubectl exec -n posthog -it $POSTHOG_WEB_POD_NAME -- python manage.py shell_plus
```

In a moment you should see the shell load and finally a message like this appear:

```
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>>
```

That means you can now type Python code and run it with PostHog context (such as models) already loaded!
For example, to see the number of users in your instance run:

```python
User.objects.count()
```

### How do I connect to Postgres?

1. Find out your Postgres password from the web pod:

    ```bash
    # First we need to determine the name of the web pod – see "How do I see logs for a pod?" for more on this
    POSTHOG_WEB_POD_NAME=$(kubectl get pods -n posthog | grep -- '-web-' | awk '{print $1}')
    # Then we can get the password from the pod's environment variables
    kubectl exec -n posthog -it $POSTHOG_WEB_POD_NAME -- sh -c 'echo The Postgres password is: $POSTHOG_DB_PASSWORD'
    ```

2. Connect to your Postgres pod's shell:

    ```bash
    # We need to determine the name of the Postgres pod (usually it's 'posthog-posthog-postgresql-0')
    POSTHOG_POSTGRES_POD_NAME=$(kubectl get pods -n posthog | grep -- '-postgresql-' | awk '{print $1}')
    # We'll connect straight to the Postgres pod's psql interface
    kubectl exec -n posthog -it $POSTHOG_POSTGRES_POD_NAME  -- /bin/bash
    ```

3. Connect to the `posthog` database:

    > You're connecting to your production database, proceed with caution!

    ```bash
    psql -d posthog -U postgres
    ```

    Postgres will ask you for the password. Use the value you found out in step 1.
    Now you can run SQL queries! Just remember that an SQL query needs to be terminated with a semicolon `;` to run.

### How do I connect to ClickHouse?

> **Tip:** Find out your pod names with `kubectl get pods -n posthog`

1. Find out your ClickHouse user and password from the web pod:

    ```shell
    kubectl exec -n posthog -it <your-posthog-web-pod> \
    -- sh -c 'echo user:$CLICKHOUSE_USER password:$CLICKHOUSE_PASSWORD'
    ```

2. Connect to the `chi-posthog-posthog-0-0-0` pod:

    ```shell
    kubectl exec -n posthog -it chi-posthog-posthog-0-0-0  -- /bin/bash
    ```

3. Connect to ClickHouse using `clickhouse-client`:

    > **Note:** You're connecting to your production database, proceed with caution!

    ```shell
    clickhouse-client -d posthog --user <user_from_step_1> --password <password_from_step_1>
    ```

### How do I restart all pods for a service?

> **Important:** Not all services can be safely restarted this way. It is safe to do this for the app/plugin server. If you have any doubts, ask someone from the PostHog team.

1. Terminate all running pods for the service:

    ```shell
    # substitute posthog-plugins for the desired service
    kubectl scale deployment posthog-plugins --replicas=0 -n posthog
    ```

2. Start new pods for the service:

    ```shell
    # substitute posthog-plugins for the desired service
    kubectl scale deployment posthog-plugins --replicas=1 -n posthog
    ```
