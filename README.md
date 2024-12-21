# Note: this repo is forked from a public Github repo to start. The intention of this is to explore an solution to let engineers of Connectbase have a way to easily analyze our application logs stored in cold archive. So we can analyze logs even if they aren't available on datadog.

# Docker-compose files for a simple uptodate

# InfluxDB

# + Grafana stack

# + Telegraf

Get the stack (only once):

```shell
# git clone this repo first
cd docker-influxdb-grafana
docker pull grafana/grafana
docker pull influxdb
docker pull telegraf
```

Run your stack:

```
docker-compose up -d
```

Show me the logs:

```
docker-compose logs
```

Stop it:

```
docker-compose stop
docker-compose rm
```

Update it:

```
git pull
docker pull grafana/grafana
docker pull influxdb
docker pull telegraf
```

If you want to run Telegraf, edit the telegraf.conf to yours needs and:

```
docker exec telegraf telegraf
```

When this compose cluster is up for the first time, it's recommanded to initialize InfluxDB and Grafana in below method:

## InfluxDB Setup

reference: https://www.influxdata.com/blog/getting-started-influxdb-grafana/

```shell
# ssh into the container of influxdb, in order to we can run below commands using influx-cli. Assuming the host machine did not install influx-cli
docker exec -it influxdb bash

# Create initial super-admin credentials, organization, bucket and the all-access security token.
influx setup --name myinfluxdb2 --host http://localhost:8086 \
  -u admin -p admin54321 -o my-org \
  -b my-bucket -t my-token -r 0 -f
```

## Grafana and influxDB connection Setup

### add datasource in Grafana UI

Open in browser http://localhost:3000/datasources

Sign in as user admin, password admin.

Click on Skip to skip the question about the new password.

In the left menu, click on the Gear icon, to open Data Sources.

Click on Add data source.

Select InfluxDB.

Replace InfluxQL with Flux in the dropdown called Query Language.

Type http://influxdb:8086/ at the URL field in the section called HTTP.

Write my-org into the Organization field in the InfluxDB Details section.

Type my-token in the Token field. (Once the save and test button is clicked, the password is hidden and replaced with configured.)

Save & Test: Success will display two green notifications (3 buckets found + Datasource updated). Please see below.

\*in the above reference article, it showed you to run a sample query against influxDB. For those who want to try, you can click into the above documentation.

## Load JSON logs in our cold archive (draft | In-progress)

### assumptions:

JSON data files has been downloaded from our code archive.

The format of our cold logging would include lines of JSON object like

```json
{
  "date": "2024-09-08T13:11:15.156Z",
  "service": "remotepricingapi",
  "host": "TCW-Prod-Web01",
  "attributes": {
    "service": "remotepricingapi",
    "logger": "com.connected2fiber.service.ARemotePricingBase",
    "host": "TCW-Prod-Web01",
    "pid": "63668",
    "attributes": {
      "dd": {
        "trace_id": "4757013253263575329",
        "span_id": "1926428052441208032",
        "service": "RemotePricingAPI",
        "env": "prod",
        "version": "web01"
      },
      "product_service_ids": "15117",
      "address_key": "2460 VETERANS BLVD|KENNER|LA|USA",
      "request_id": "1522506-1373194-e"
    },
    "journald": {
      "_PID": "63668",
      "_CAP_EFFECTIVE": "1fffffffff",
      "PRIORITY": "6",
      "_SYSTEMD_CGROUP": "/system.slice/remotepricingapi.service",
      "_GID": "0",
      "_MACHINE_ID": "5ae61cd0378c4570bd8f81793f37e417",
      "_COMM": "java",
      "_STREAM_ID": "9d7a4525ea06441eaf2060f4fabe596e",
      "_BOOT_ID": "1ac33330e8904db39bd8ff7e4fcb60fd",
      "_SYSTEMD_UNIT": "remotepricingapi.service",
      "_TRANSPORT": "stdout",
      "_CMDLINE": "/usr/bin/java -XX:MaxDirectMemorySize=1G -XX:+ExitOnOutOfMemoryError -Xms4048m -Xmx16384m -Dlog4j2.formatMsgNoLookups=true -javaagent:/data/dd-java-agent.jar -Ddd.service=RemotePricingAPI -Ddd.env=prod -Ddd.version=web01 -Ddd.profiling.enabled=true -XX:FlightRecorderOptions=stackdepth=256 -Delastic.apm.service_name=RemotePricingAPI -Delastic.apm.server_urls=http://192.168.89.21:8200 -Delastic.apm.application_packages=org.fiber -jar -Dspring.config.location=/data/connectedworld/environments/remote_pricing_api/remotePricing.properties build/libs/RemotePricingAPI-1.0.0.jar",
      "SYSLOG_IDENTIFIER": "java",
      "_SYSTEMD_SLICE": "system.slice",
      "SYSLOG_FACILITY": "3",
      "_UID": "0",
      "_HOSTNAME": "TCW-Prod-Web01",
      "_EXE": "/usr/lib/jvm/java-11-openjdk-11.0.17.0.8-2.el7_9.x86_64/bin/java"
    },
    "env": "prod",
    "version": "1.0",
    "timestamp": "2024-09-08T13:11:15.156+0000",
    "status": "INFO"
  },
  "_id": "AZHRwpH8AADT75RMhy1kXwAD",
  "source": "java",
  "message": "[ACE-Core-Flow] Notify of completion",
  "status": "info",
  "tags": [
    "service:remotepricingapi",
    "source:java",
    "datadog.submission_auth:api_key",
    "availability-zone:eastus",
    "client_id:85cc4d89-154e-42e9-94f2-d205503a298a",
    "cloud_provider:azure",
    "confidentiality:unknown",
    "criticality:critical",
    "env:prod",
    "name:tcw-prod-web01",
    "operating_system:linux",
    "region:eastus",
    "resource_group:tcw-production",
    "size:standard_f64s_v2",
    "subscription_id:1a6072ad-5b67-4f24-b4bc-5b4d90bddb0d",
    "subscription_name:main_azure_invoice_based_subscription",
    "tenant_name:7d3e3b34-ea78-42b4-98aa-1ab2a2ddb079"
  ]
}
```
