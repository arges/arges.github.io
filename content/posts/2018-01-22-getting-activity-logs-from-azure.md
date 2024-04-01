---
type: post
title: getting activity logs from azure
date: '2018-01-22T11:26:32-0600'
author: arges
tags:
- azure
- python
modified_time: '2018-01-22T11:26:36-0600'

---

I was tasked with looking into how to connect to the Azure API using python.
Below are some of my setup notes for getting this working.

First install the az client. Follow this link to get things setup on your platform:
`https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest`

In order to create our client you can create a json file that contains the
necessary credentials. This can be consumed by the python library.
```
az login
az ad sp create-for-rbac --sdk-auth > mycredentials.json
```
If you get a `Role assignment creation failed` error you'll need to ensure you
have proper permissions from your Azure administrator.

Create virtualenv and install azure python libraries:
```
python3 -m virtualenv env
source env/bin/activate
pip install --pre azure
```

Putting it all together we can go from using our credentials file, getting a
client object, querying and printing the results.

First create a python file using the below code:
```python
from azure.common.client_factory import get_client_from_auth_file
from azure.monitor import MonitorClient
import datetime

# Get a client for Monitor
client = get_client_from_auth_file(MonitorClient)

# Generate query here
today = datetime.datetime.now().date()
filter = "eventTimestamp ge {}".format(today)
select = ",".join([
    "eventTimestamp",
    "eventName",
    "operationName",
    "resourceGroupName",
])

# Grab activity logs
activity_logs = client.activity_logs.list(
    filter=filter,
    select=select
)

# Print the logs
for log in activity_logs:
    print(" ".join([
        str(log.event_timestamp),
        str(log.resource_group_name),
        log.event_name.localized_value,
        log.operation_name.localized_value
    ]))
```

The function `get_client_from_auth_file` expects the environment variable
`AZURE_AUTH_LOCATION` to have the path of the `credentials.json` file so it can
create the client object.

So first set the variable:
```
export AZURE_AUTH_LOCATION=./mycredentials.json
```

Then execute the python:
```
python azure.py
```

Then you can see activity logs!
