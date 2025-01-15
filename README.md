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

## InfluxDB Setup （for information only)

(skip this part, because inititialization has been setup in `env.influxdb` file)
reference: https://www.influxdata.com/blog/getting-started-influxdb-grafana/

```shell
# ssh into the container of influxdb, in order to we can run below commands using influx-cli. Assuming the host machine did not install influx-cli
docker exec -it influxdb bash

# Create initial super-admin credentials, organization, bucket and the all-access security token.
influx setup --name myinfluxdb2 --host http://localhost:8086 \
  -u admin -p admin54321 -o my-org \
  -b my-bucket -t my-token -r 0 -f
```

Note: as this project is for local usage. These default values will be used in other setups. If we need to make them configurable, we can do in the future.

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

#### Case 1: connect influxDB to Grafana using InfluxQL

1. according to https://docs.influxdata.com/influxdb/v2/tools/grafana/?t=InfluxQL#installed-a-new-influxdb-instance, we need to do two things first: enabled influxDB V1-compatible authentication credentials, and manually create DBRP mappings. 

Let's get in influxDB instance: `docker exec -it influxdb bash`

a. view existing V1 authorizations: `influx v1 auth list`. You are supposed to see results like below 
```shell
root@77b06e013f79:/# influx v1 auth list
ID      Description     Username        v2 User Name    v2 User ID      Permissions
```
b. create a V1 authorization

to create V1-compatible credentials, we need to know the bucket ID of our default bucket `my-bucket` first. 
execute `influx bucket list` in the bash shell inside the catainer.

then you would see something like this (actual values are different than the example)
```shell
root@77b06e013f79:/# influx bucket list
ID                      Name            Retention       Shard group duration    Organization ID         Schema Type
ecb6cdeb10297804        _monitoring     168h0m0s        24h0m0s                 3da3c410fb4f8edf        implicit
6e27309b92a7f9e0        _tasks          72h0m0s         24h0m0s                 3da3c410fb4f8edf        implicit
72c336c6fdf0150a        my-bucket       infinite        168h0m0s                3da3c410fb4f8edf        implicit
```

In this case, we will use `72c336c6fdf0150a` in the rest of steps.

Use the influx v1 auth create command to grant read/write permissions to specific buckets. Provide the following:

bucket IDs to grant read or write permissions to
new username
new password (when prompted)

```shell
influx v1 auth create \
  --read-bucket 72c336c6fdf0150a \
  --write-bucket 72c336c6fdf0150a \
  --username admin
```

After double-typing the credential, we finished creating V1 compatible authorization

```shell
root@77b06e013f79:/# influx v1 auth create \
  --read-bucket 72c336c6fdf0150a \
  --write-bucket 72c336c6fdf0150a \
  --username admin
? Please type your password **********
? Please type your password again **********
ID                      Description     Username        v2 User Name    v2 User ID              Permissions
0e3622b6fa3da000                        admin           admin           0e361a8162da6000        [read:orgs/3da3c410fb4f8edf/buckets/72c336c6fdf0150a write:orgs/3da3c410fb4f8edf/buckets/72c336c6fdf0150a]
```

2. view and create InfluxDB DBRP mappings 

[quoting : https://docs.influxdata.com/influxdb/v2/tools/grafana/?t=InfluxQL#view-and-create-influxdb-dbrp-mappings]
> When using InfluxQL to query InfluxDB, the query must specify a database and a retention policy. InfluxDB DBRP mappings associate database and retention policy combinations with InfluxDB 2.7 buckets.
> DBRP mappings do not affect the retention period of the target bucket. These mappings allow queries following InfluxDB 1.x conventions to successfully query InfluxDB 2.7 buckets.

a. view existing DBRP mappings
`influx v1 dbrp list`

should be like 
```shell
root@77b06e013f79:/# influx v1 dbrp list
ID      Database        Bucket ID       Retention Policy        Default Organization ID

VIRTUAL DBRP MAPPINGS (READ-ONLY)
----------------------------------
ID                      Database        Bucket ID               Retention Policy        Default Organization ID
ecb6cdeb10297804        _monitoring     ecb6cdeb10297804        autogen                 true    3da3c410fb4f8edf
6e27309b92a7f9e0        _tasks          6e27309b92a7f9e0        autogen                 true    3da3c410fb4f8edf
72c336c6fdf0150a        my-bucket       72c336c6fdf0150a        autogen                 true    3da3c410fb4f8edf
```

b. create a DBRP mapping

> Use the influx v1 dbrp create command command to create a DBRP mapping. Provide the following:
> database name
> retention policy name (not retention period)
> bucket ID
> (optional) --default flag if you want the retention policy to be the default retention policy for the specified database

```shell
influx v1 dbrp create \
  --db my-bucket \
  --rp example-rp \
  --bucket-id 72c336c6fdf0150a \
  --default
```

after finishing the above sections (a and B)

we can configure InfluxDB as a data source in Grafana, using the `influxQL` option of Query Language. (see step 3 of https://docs.influxdata.com/influxdb/v2/tools/grafana/?t=InfluxQL#configure-your-influxdb-connection) 

Note: Unlike the screenshot from the step 3 link above, our verified way to configure influxDB using InfluxQL in Grafana: turn off any button in `Auth` section,  do not create custom Authentication Header, and put `my-bucket` in database field, and put our default user `admin` and password `admin54321` , and chose HTTP method `GET`, others can be default. 

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

### preparations

data clean up:

to get rid of JSON data that's too long (one json line with more than 100000 charactoers), we need to filter them.

e.g.

```shell
awk 'length($0) <= 100000' \
    ./data/json-logging/archive_130546.7614.zzJc7_uNRFS5A6LiamyP_g.json \
    > ./data/json-logging/simplified_archive.json
```

### steps

1. move those JSON files into folder `./data/json-logging`
2. make sure the `telegraf.conf` has properly set up sections `[[inputs.tail]]` and `[[outputs.influxdb_v2]]`
3. start the cluster: `docker-compose up -d`
