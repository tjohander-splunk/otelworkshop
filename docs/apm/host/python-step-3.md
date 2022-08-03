
---

* Each step should be performed in a separate terminal window.

* Make sure your Ubuntu environment was prepared properly as described in the **Preparation** section.  

## Validate spans are being sent

**Open a new terminal window** in your Linux instance to check OpenTelemetry Collector Statistics to see that spans are being sent.

```bash
lynx localhost:55679/debug/tracez
```
will show the metrics and spans being gathered and sent by the Collector.  

Lynx is a text browser that was installed during with the `setup-tools`. Enabling a web browser to access your environment will allow for a full web GUI.  

![zpagaes](../../../images/06-zpages.png)

### APM Dashboard

Traces / services will now be viewable in the APM dashboard. A new service takes about 90 seconds to register for the first time, and then all data will be available in real time.  

The `Environment` pulldown will let you see the APM map associated with your individual environment that you set with your initials if this was done earlier.  
  
Additionally span IDs will print in the terminal where flask-server.py is running. You can use ++ctrl+c++ to stop the requests and server any time.  

The Python server application will be called: `py-otel-flask-server`  and the client will be called `py-otel-client`.  

Navigate to `Splunk Overvability -> APM`

![image](../../images/07-apm.png)

Service map of this python demo  

![image](../../images/08-python.png)

Click on one of the peaks in the grey graph within "Services By Latency (P90)" on the right hand side and then click the trace to see spans. Also try out **Tag Spotlight** to see how application operations are broken down in a granular way. You can also try the **Tags** menu on top to search for a single trace or group of traces by key:value.

![image](../../images/09-pythontraces.png)  
![image](../../images/10-pythonspans.png)  

To learn more about traces and spans [see the Splunk APM documentation](https://docs.splunk.com/Observability/apm/terms-concepts/traces-spans.html#apm-traces-spans)

## Where is the OpenTelemetry Instrumentation?

The `run-server.sh` and `run-client.sh` scripts set up the environment variables for OpenTelemetry and invoke the Python auto instrumentation:  

`spluk-py-trace` is the auto instrumenting function that runs Python3 with the instrumentation that automatically emits spans from the Python app. No code changes are necessary. Splunk Observability Cloud has a `Data Setup` Wizard to guide through instrumentation setup.

OpenTelemetry repo for Python is [here](https://github.com/signalfx/splunk-otel-python).

!!! important
    Leave the Flask server running you'll need need this process for the next client examples in the workshop.