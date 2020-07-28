# balena-blocks/transport

Intelligently connect data sources with data sinks in block-based balena applications.
The `transport` block is a docker image that runs [telegraf](https://www.influxdata.com/time-series-platform/telegraf/) and code to find other services running on the device, and intelligently connect them.

## Features

- Automatically finds HTTP data sources on the device
- Automatically finds supported data storage services on the device
- Configurable HTTP listener
- Configurable external HTTP data source(s)
- Configurable HTTP output
- Device metric support (CPU and Memory usage)

## Usage

#### docker-compose file
To use this image, create a container in your `docker-compose.yml` file as shown below:

```yaml
version: '2.1'

services:
  transport:
    image: phildwilson/balenalabs-transport:raspberrypi4-64
    restart: always
    labels:
      io.balena.features.balena-api: '1' # necessary to discover services
    privileged: true # necessary to change container hostname
    ports:
      - "8080" # only necessary if using ExternalHttpListener (see below)
```

You can also set your `docker-compose.yml` to build a `dockerfile.template` file, and use the build variable `%%BALENA_MACHINE_NAME%%` so that the correct image is automatically built for your device type (see [supported devices](#Supported-devices)):

*docker-compose.yml:*
```yaml
version: '2'

volumes:
  settings:                          # Only required if using PERSISTANT flag (see below)

services:

  transport:
    build: ./
    restart: always
    labels:
      io.balena.features.balena-api: '1' # necessary to discover services
    privileged: true # necessary to change container hostname
    ports:
      - "8080" # only necessary if using ExternalHttpListener (see below)
```
*dockerfile.template*

```dockerfile
FROM phildwilson/balenalabs-transport:%%BALENA_MACHINE_NAME%%
```

## Supported devices
The `transport` block has been tested to work on the following devices:

| Device Type  | Status |
| ------------- | ------------- |
| Raspberry Pi 3b+ | ✔ |
| Raspberry Pi 4 | ✔ |
| Intel NUC | coming |
| Generic AMD64 | coming |
</br>

## Data Sources
### Internal HTTP
This type of data source runs it's own HTTP server and provides data readings as `json` strings. The service must expose port `7575` like this:

```yaml
  sensor:
    build: ./sensor
    expose:
      - '7575'
```
The `transport` block will find this service and configure telegraf to periodically pull from it via HTTP.

### MQTT
By adding an MQTT broker to an application, you can push data into the `transport` block. Add your broker such as:

```yaml
mqtt:
    image: arm32v6/eclipse-mosquitto
    ports:
      - "1883:1883"
    restart: always
```
As long as you call the service `mqtt` the `transport` block will automatically find it and configure telegraf to pull data from the broker. Ensure the data is formatted as `json` strings. Telegraf will be configured to only pull from the `sensors` topic, so any other data you may wish to put onto the MQTT broker will not be stored (e.g. control or signalling messages).

*Example code:*
```python
client = mqtt.Client("1")
client.connect("localhost")

while(True):
    value = GetReading() # code omitted for brevity
    client.publish("sensors",json.dumps(value))
    time.sleep(5)
```

### External HTTP Pull
This type of source is pulled from a provide via the internet. It is enabled by adding an environment variable to the `transport` service called `EXTERNAL_HTTP_PULL_URL` and setting it to the URL of the source:

![alt text](https://i.ibb.co/z4MVcxw/External-HTTPConfig.jpg "balenaCloud device service variable")

Setting the vaiable `EXTERNAL_HTTP_PULL_NAME` (as above) allows you to rename the resulting data source, otherwise it will appear in your data sinks (see below) as `inputs.http`

### External HTTP PUSH
This type of data source pushes to your device. It is configured by enabling a built-in HTTP listener with the environment variable `ENABLE_EXTERNAL_HTTP_LISTENER` set to `1`:

![alt text](https://i.ibb.co/rwYbXWj/External-HTTPListener-Config.jpg "balenaCloud device service variable")

Again, the resulting data source can be given a custom name (as above) by setting the `EXTERNAL_HTTP_LISTENER_NAME` variable.

### Device Metrics
This data source provides the in-built telegraf metrics for the CPU and Memory usage of the device. It is enabled by setting the environment variable `ENABLE_DEVICE_METRICS` to `1`.

This data source is useful for testing `transport` or simply to allow device resource monitoring as part of your application.

## Data Sinks
### InfluxDB
Adding an influx timeseries database to your application will cause the `transport` block to configure telegraf to push data into it. You must name the service `influxdb` for it to be automatically discovered, such as:

```yaml
influxdb:
    image: influxdb@sha256:73f876e0c3bd02900f829d4884f53fdfffd7098dd572406ba549eed955bf821f
    container_name: influxdb
    restart: always
```

By default the database used will be called `balena`, however you can set a custom database name by setting the `INFLUXDB_DB` environment variable.

### HTTP Data Sink
This data sink will send the data to a URL specified with the environment variable `HTTP_PUSH_URL`. 

### Azure Monitor
This data sink pushes data into the Azure Monitor service, using an [Azure Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/cloudservices) account. In order to use this sink, login (or create) to your Microsoft Azure account, create an Application Insights resource and copy the instrumentation key. Guide here:
https://docs.microsoft.com/en-us/azure/azure-monitor/app/create-new-resource

Place this key into an environment variable on your balena device called `APPLICATION_INSIGHTS_KEY`. 

You can now view the data by pointing Azure Monitor to your Application Insights account and charting the correct metrics:

![alt text](https://i.ibb.co/6r61Ykg/azure.jpg "Azure AppInsights")

## Customisation
### Extend image configuration

By default the `transport` block creates a telegraf configuration file from the combination of discovered services and device environment variables. However for custom configurations you can overload the `CMD` directive, as such:

*dockerfile.template*
```Dockerfile
FROM balenaplayground/balenalabs-browser:%%BALENA_MACHINE_NAME%%

COPY customTelegraf.conf .

CMD ["--config customTelegraf.conf"]
```

This will stop the auto-wiring code from running and cause telegraf to be run purely from the supplied configuration file.





