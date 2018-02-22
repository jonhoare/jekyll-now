When it comes to monitoring, I'm a big fan of using [datadog](http://www.datadoghq.com) to consume all of the metrics from my infrastructure and applications, allowing me to build custom dashboards and alerts that show me all the parts of my system that I care about, particularly **NServiceBus**.
This isn't a blog about the in's and out's of datadog, that's for another blog post yet to come, but instead how you can have datadog read your metrics from NServiceBus.

Datadog has a wealth of built in checks and integrations that can be enabled out of the box but, if like me, your application relies heavily on the use of [NServiceBus](https://particular.net), especially using MSMQ as the underlying transport, you may be disappointed to find that there is no out of the box integration to gather these metrics.

Sure you could configure your Datadog agent to perform a `wmi_check` to query the `Win32_PerfFormattedData_MSMQ_MSMQService` class and collect/send the `MessagesInQueue` metric, but other than getting the current queue lengths this isn't a particularly useful metric.

I'm going to show you how you can make use of the [NServiceBus.Metrics.ServiceControl](https://www.nuget.org/packages/NServiceBus.Metrics) package to send metrics about your NServiceBus Host to the new [Particular ServiceControl Monitoring Endpoint](https://docs.particular.net/servicecontrol/monitoring-instances), and have our datadog agent collect these metrics by querying the ServiceControl Monitoring API.

**DISCLAIMER: Particular state the following in their documentation and so you use this solution at your own risk!**

> The ServiceControl HTTP API is designed for use by [Service Pulse](https://docs.particular.net/servicepulse) only and may change at any time. Use of this HTTP API for other purposes is discouraged.

# Install/Configure the ServiceControl Monitoring Endpoint
If you are not already using ServiceControl and ServicePulse to get an insight into your NServiceBus applications, you are going to need to go ahead and install these from [Particulars Download Page](https://particular.net/downloads).

## Install ServiceControl
First off install ServiceControl as you will need this configured before setting up ServicePulse. You can go ahead and use the default options to install, and once the installation is complete, launch the ServiceControl Management Utility (This is the default option at the end of the installation).

### Configure Service Control Instance
Go ahead and add a new `ServiceControl Instance`, setting or using the default configuration options. Make sure you specify your Transport, in our case this is `MSMSQ` and set the `Audit Forwarding` option.

### Configure Monitoring Instance
Go ahead and add a new `Monitoring Instance`, setting or using the default configuration options. Make sure you specify your Transport, in our case this is `MSMSQ`.

## Install ServicePulse
Ensure that both of your ServiceControl instances are up and running, and then go ahead and install ServicePulse. you can use the default settings, or change them to reflect your ServiceControl configuration. Once the installation has finished, you will be able to access the ServicePulse Web UI at [http://localhost:9090](http://localhost:9090).

# Add NServiceBus.Metrics.ServiceControl package to NServiceBus Hosts

Time to get into some code. Open up your NServiceBus Host code and add the `NServiceBus.Metrics.ServiceControl` nuget package

```
Install-Package NServiceBus.Metrics.ServiceControl
```
In your EndpointConfiguration, add the following code which will configure your endpoint to send metrics to the `particular.monitoring` endpoint.

```c#
const string SERVICE_CONTROL_METRICS_ADDRESS = "particular.monitoring";

var endpointName = "MyEndpoint";
var machineName = $"{Dns.GetHostName()}.{IPGlobalProperties.GetIPGlobalProperties().DomainName}";
var instanceIdentifier = $"{endpointName}@{machineName}";

var metrics = endpointConfiguration.EnableMetrics();

metrics.SendMetricDataToServiceControl(
    serviceControlMetricsAddress: SERVICE_CONTROL_METRICS_ADDRESS,
    interval: TimeSpan.FromSeconds(10),
    instanceId: instanceIdentifier);
```

## Testing out the Metrics with ServicePulse
Now run up your Endpoint and let it start handling messages and head back over to the ServicePulse Web UI at [http://localhost:9090](http://localhost:9090), go to the monitoring page and you should see your endpoints metrics coming through to ServicePulse.

Assuming all is good, we can go ahead and create a custom datadog agent check to query the ServiceControl Monitoring API at [http://localhost:33633](http://localhost:33633).

For more information about custom datadog agent checks visit the [datadog agent check documentation](https://docs.datadoghq.com/agent/agent_checks/).

# Writing a Datadog agent check
I am going to assume that you installed the datadog agent in the default location for all of this.

## Create the yaml configuration file
Create a file `service_control.yaml.sample` at `C:\ProgramData\Datadog\conf.d` and add the following lines.
This is the file that the agent configuration manager uses to configure the check. You can specify the ServiceControl Monitoring instance host, which defaults to `localhost`, the port which defaults to `33633` and any tags that you want all instances to be tagged with. 

```yaml
init_config:

instances:
  - host: "."
# - port: 33633
# - tags:
#    - "mytag"
#    - "tagKey:tagValue"
```

## Create the agent check
Now create a new file `service_control.py` at `C:\Program Files\Datadog\Datadog Agent\checks` and add the following code.

*Note: My python is not brilliant and certainly not my strong point so please excuse the code. There are definitely improvements that I could make and it has been hacked together for the purpose of this demo.*

```python
import requests

from checks import AgentCheck
from hashlib import md5

class ServiceControlMonitoring(AgentCheck):
    _base_url = ""
    _params = "history=1"

    def check(self, instance):
        host = instance.get("host", "localhost")
        port = instance.get("port", "33633")
        tags = instance.get("tags", [])

        if (host == "."):
            host = "localhost"

        self._base_url = "http://{0}:{1}/monitored-endpoints".format(host, port)
        
        url = self.get_url()
        aggregation_key = md5(url).hexdigest()

        try:
            http_result = requests.get(url, timeout=5)
            self.process_endpoints(http_result.json(), tags)
        except requests.exceptions.Timeout as e:
            self.timeout_event(url, timeout, aggregation_key)
    
    def timeout_event(self, url, timeout, aggregation_key):
        self.event({
            'timestamp': int(time.time()),
            'event_type': 'service_control_monitoring_check',
            'msg_title': 'Service Control Monitoring timeout',
            'msg_text': '%s timed out after %s seconds.' % (url, timeout),
            'aggregation_key': aggregation_key
        })
    
    def get_url(self, endpoint_name=""):
        return "{0}/{1}?{2}".format(self._base_url, endpoint_name, self._params)

    def process_endpoints(self, data, constant_tags):
        for endpoint in data:
            endpoint_name = endpoint["name"]

            tags = []
            tags.extend(constant_tags)
            tags.append("endpoint:{0}".format(endpoint_name))

            url = self.get_url(endpoint_name)
            http_result = requests.get(url, timeout=5)
            self.process_endpoint_data(http_result.json(), tags)

    def process_endpoint_data(self, data, constant_tags):        
        self.process_data(data["instances"], "name", "instance", constant_tags)
        self.process_data(data["messageTypes"], "typeName", "messagetype", constant_tags)

    def process_data(self, data, name_attr, metric_type, constant_tags):
        for obj in data:
            name = obj[name_attr]
            
            tags = []
            tags.extend(constant_tags)
            tags.append("{0}:{1}".format(metric_type, name))

            self.send_metrics(obj["metrics"], metric_type, tags)

    def send_metrics(self, metrics, metric_type, tags):
        if ("processingTime" in metrics):
            self.gauge("particular.sc.{0}.processtime".format(metric_type),  metrics["processingTime"]["average"], tags=tags)

        if ("criticalTime" in metrics):
            self.gauge("particular.sc.{0}.criticaltime".format(metric_type), metrics["criticalTime"]["average"], tags=tags)
            
        if ("retries" in metrics):
            self.gauge("particular.sc.{0}.retries".format(metric_type), metrics["retries"]["average"], tags=tags)
            
        if ("queueLength" in metrics):
            self.gauge("particular.sc.{0}.queuelength".format(metric_type), metrics["queueLength"]["average"], tags=tags)
            
        if ("throughput" in metrics):
            self.gauge("particular.sc.{0}.throughput".format(metric_type), metrics["throughput"]["average"], tags=tags)

if __name__ == '__main__':
    check, instances = HTTPCheck.from_yaml('C:/ProgramData/datadog/conf.d/service_control_monitoring.yaml')
    for instance in instances:
        host = instance.get("host", "localhost")
        port = instance.get("port", "33633")

        url = "http://{0}:{1}/monitored-endpoints?history=1".format(host, port)

        print "\nRunning the check against url: %s" % (url)
        check.check(instance)
        if check.has_events():
            print 'Events: %s' % (check.get_events())
        print 'Metrics: %s' % (check.get_metrics())
```

This agent check will be run every 15 seconds, which is the default datadog behaviour, and will then make a HTTP GET request to your ServiceControl Monitoring API at `http://instance:port/monitored-endpoints?history=1`.
The parameter `history=1` will return the `Average` and all the metrics recorded over the past `1 minute`.

For each endpoint that is returned from the API a new request is made to the `http://instance:port/monitored-endpoints/EndpointName?history=1`.

From this result we are gathering the `Average` value from each metric from the past 1 minute period and submitting this as a gauge metric to datadog, tagged with the `endpoint`, `instance`, and the `messageType`.

The metrics that are available in datadog are
* `particular.sc.instance.processtime`
* `particular.sc.instance.criticaltime`
* `particular.sc.instance.queuelength`
* `particular.sc.instance.retries`
* `particular.sc.instance.throughput`
* `particular.sc.messageType.processtime`
* `particular.sc.messageType.criticaltime`
* `particular.sc.messageType.retries`
* `particular.sc.messageType.throughput`

## Enable the Agent Check

Open up the Datadog Agent Configuration Manager and scroll down the available checks to find our new `Service Control` check. Set any of the parameters that you require and hit `Save` and the finally hit the `Enable` button.

Restart the Agent by clicking `Actions` -> `Restart` and then head over to the Agent Status `Logs and Errors` -> `Agent Status` and look for the checks section. You should see a section for our `Service Control` check and you should see some metrics that have been collected.

# View the metrics in Datadog
It takes a couple of minutes for your metrics to start appearing in your datadog instance so please be patient.

Head over to the Datadog Web UI and check the `Metrics` -> `Summary`, search for `particular` metrics and you should see our new metrics coming through.

Create a new dashboard, or open an existing one, add a `TopList` and select to display the `particular.sc.instance.throughput`, restrict the results to certain tags if required, and average the results by `instance`. This will display the throughput metric that we have collected, breaking it down by the instance name.

Eg If you are running `myservice` on hosts `myhost1` and `myhost2` you should see metrics for
* `myservice@myhost1`
* `myservice@myhost2`

***Please remember that Particular DO NOT endorse accessing the ServiceControl Monitoring API like this and if you do so you do so at your own risk and acknowledge the risks that the API is subject to change.***

