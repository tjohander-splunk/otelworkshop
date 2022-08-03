
---

* Each step should be performed in a separate terminal window.

* Make sure your Ubuntu environment was prepared properly as described in the **Preparation** section.  

## Run Python Client Application

**Open a new terminal window** in your Linux instance and run the Python client to sent POST requests to the Flask server:  

Run the client Python app via the `splunk-py-trace` command to send requests to the Flask server:  

!!! important
    If you are doing this workshop as part of a group, before the next step, add your initials do the APM environment:
    edit the `run-client.sh` script below and add your initials to the environment i.e. change:  
    `export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=apm-workshop`  
    to    
    `export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=sjl-apm-workshop`  

```bash
cd ~/otelworkshop/host/python
source run-client.sh
```

The `python-requests.py` client will make calls to the flask server with a random short sleep time.  
You can stop the requests with ++ctrl+c++