
---

* Each step should be performed in a separate terminal window.

* Make sure your Ubuntu environment was prepared properly as described in the **Preparation** section.  

## Configure Environment Variables for Otel and Run Python Flask Server

**Open the first terminal window** in your Linux instance and set up environment and run Python Flask server using auto-instrumentation:

!!! important
    If you are doing this workshop as part of a group, before the next step, add your initials do the APM environment:
    edit the `run-server.sh` script below and add your initials to the environment i.e. change:  
    `export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=apm-workshop`  
    to    
    `export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=sjl-apm-workshop`  

```bash
cd ~/otelworkshop/host/python
source run-server.sh
```

You will see the server startup text when this is run.