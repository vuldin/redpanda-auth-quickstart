This guide will cover some initial steps using Redpanda centered around authentication via SASL within a test environment:

- starting up a single-node cluster and Redpanda Console (with `docker compose`)
- connecting a simple librdkafka client
- using the HTTP proxy endpoint
- enabling SASL authentication in Redpanda
- configuring authentication for Redpanda Console, the librdkafka client, and the HTTP proxy endpoint

> Note: This environment is not meant for production! Please see [this page](https://docs.redpanda.com/docs/deploy/deployment-option/self-hosted/manual/production/production-deployment/) for more details on how to configure for production.

## Prerequisites

- Docker compose (see [here](https://docs.docker.com/compose/install/))
- building a client with librdkafka (see [here](https://github.com/confluentinc/librdkafka/))
- clone this repo (all steps will start from the root directory)

## Steps


### Start Redpanda

We'll start with a docker compose file that is almost identical to what is found in the quickstart [here](https://docs.redpanda.com/docs/get-started/quick-start/):

```
docker compose up
```

> Note: If Redpanda fails to start, see [here](#redpanda-fails-to-start).

This will start a single-node Redpanda cluster along with Redpanda Console. This initial environment will have no authentication (via SASL), and it will be added once we verify connectivity with various clients.

### Connecting with various clients

### Console

You can view Redpanda Console at http://localhost:8080 .

#### rpk

`rpk` is the Redpanda CLI, and is installed by default when you install Redpanda.

We will install `rpk` on the local system in order to walk through configuring an external client. Follow [these steps](https://docs.redpanda.com/docs/get-started/rpk-install/) to get `rpk` installed locally.

By default, `rpk` uses the config file `/etc/redpanda/redpanda.yaml` (this is the same config file that Redpanda uses). The `rpk` section controls the CLI's connection to Redpanda. The default config (above) provides no changes to how `rpk` will connect to Redpanda. This means `rpk` will attempt to connect to the Kafka API at `localhost:9092`, and the admin API at `localhost:9644`.

Create a topic:

```
rpk topic create continents
```

Produce some data:

```
rpk topic produce -k from-rpk continents
```

Now type the name of a continent and hit enter. Each time you hit enter a new message gets sent to the topic. Press `Ctrl+C` to stop.

You can also consume data with `rpk`:

```
rpk topic consume -g rpk-group continents
```

This will start reading at the initial offset and consume until the end of the data. If you stop and then restart the client, it will continue consuming data where it previously left off. See `rpk topic consume --help` for configuration details. Press `Ctrl+C` to stop.

#### librdkafka

First you will need to build the producer app:

```
cd librdkafka-example
make producer
./producer config.ini
```

This will produce a handful of messages to a the `continents` topic. You can then consume this data with a consumer:

```
make consumer
./consumer config.ini
```

You will see the client eventually print the same data from the topic, and then loop to wait for updates. Close the client with `Ctrl-C`.

#### HTTP proxy

The HTTP proxy provides a REST API on top of Redpanda. More details are [here](https://docs.redpanda.com/docs/develop/http-proxy/).

Produce a new event into the same topic as the producer client, only using the HTTP proxy API:

```
curl -s \
  -X POST "http://localhost:8082/topics/continents" \
  -H "Content-Type: application/vnd.kafka.json.v2+json" \
  -d '{"records":[{"key": "from-proxy","value":"North America"}]}'
```

There are several possible steps clients can take in order to consume as part of a consumer group (CG):

1. create the CG
2. retrieve an instance ID for your consumer within the CG
3. subscribe your consumer to a topic with your instance ID
4. retrieve events from the topic
5. commit offsets to the CG

More details on this process and how it relates to the HTTP proxy API are [here](https://docs.redpanda.com/docs/develop/http-proxy/). In the following steps, we will focus creating the CG, retrieving the instance ID, subscribing to a topic, and reading data.

Create the CG and retrieve your client's instance ID:

```
curl -s -X POST "http://localhost:8082/consumers/proxy-group" \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  -d '{
  "format":"json",
  "name":"continents-consumer",
  "auto.offset.reset":"earliest",
  "auto.commit.enable":"false",
  "fetch.min.bytes": "1",
  "consumer.request.timeout.ms": "10000"
}'
```

Expected output:

```
{"instance_id":"continents-consumer","base_uri":"http://localhost:8082/consumers/proxy-group/instances/purchases"}
```

Subscribe to the topic with your instance ID:

```
curl -s -o /dev/null -w "%{http_code}" \
  -X POST \
  "http://localhost:8082/consumers/proxy-group/instances/continents-consumer/subscription"\
  -H "Content-Type: application/vnd.kafka.v2+json" \
  -d '{
  "topics": [
     "continents"
  ]
}'
```

Expected output:

```
204
```

Now read the messages:

```
curl -s "http://localhost:8082/consumers/proxy-group/instances/continents-consumer/records?timeout=1000&max_bytes=100000" \
  -H "Accept: application/vnd.kafka.json.v2+json"
```

### Adding authentication

Authentication can be enabled through the following steps.

Enable authorization in the cluster so that authenticated users will be able to access resources:

```
rpk cluster config set enable_sasl true
```

Designate a superuser by adding its name to the `superusers` list:

```
rpk cluster config set superusers ['admin']
```

> Note: You may need to escape the `[` and `]` symbols by prefixing with a `\`, depending on your shell.

The above only designates a superuser, so now we need to create that user:

```
rpk acl user create admin -p your_password --mechanism SCRAM-SHA-512
```
Make sure to replace `your_password` with a password of your choosing.

### Updating connections to use SASL

We need to make sure the Redpanda Console, `rpk`, the librdkafka client, and HTTP proxy requests can connect to Redpanda using SASL authentication.

#### Console 

For Console, merge the following changes into `console-config/redpanda-console-config.yaml`:

```
kafka:
  sasl:
    enabled: true
    username: admin
    password: your_password
    mechanism: SCRAM-SHA-512
```

Make sure to keep the existing configuration (just add this to the `kafka` section), and also replace `your_password` with the password you chose when creating the admin user.

#### rpk

For `rpk`, edit `/etc/redpanda/redpanda.yaml` and merge the following lines into the `rpk` section:

```
rpk:
  kafka_api:
    sasl:
      user: admin
      password: your_password
      type: SCRAM-SHA-512
```

Make sure to keep the existing configuration (just add this to the `rpk` section), and also replace `your_password` with the password you chose when creating the admin user.

#### librdkafka

For librdkafka, edit `librdkafka-example/config.ini` and merge the following lines into the `default` section:

```
sasl.mechanisms=SCRAM-SHA-512
sasl.username=admin
sasl.password=your_password
```

Make sure to keep the existing configuration (just add this to the `default` section), and also replace `your_password` with the password you chose when creating the admin user.

Also make sure to change the `security.protocol` line to the following value:

```
security.protocol=sasl_plaintext
```

#### HTTP proxy

For HTTP proxy, edit `redpanda-config/redpanda.yaml` and merge the following lines into the `pandaproxy_client` section:

```
pandaproxy_client:
  scram_username: admin
  scram_password: your_password
  sasl_mechanism: SCRAM-SHA-512
```

Make sure to keep the existing configuration (just add this `pandaproxy_client` section), and also replace `your_password` with the password you chose when creating the admin user.

### Connect via SASL

Restart the servers to make sure Redpanda and Console pick up all the configuration changes. Now you can repeat the steps in [this section](#connecting-with-various-clients), and this time all connections will be authenticated.

## Cleanup

Shutdown the containers with the following command:

```
docker compose down
```

If you have previously followed this guide, then you may want to return to the initial state. This means 1) restoring files and 2) deleting the data directory:

```
git restore -- . && git clean -df
sudo rm -r redpanda-data/*
```

## Common Issues

### Redpanda fails to start

If Redpanda fails to start, check the logs for the following error message:

```
redpanda          | ERROR 2023-04-15 01:09:04,585 [shard 0] main - application.cc:371 - Failure during startup: std::__1::system_error (error system:13, open: Permission denied)
```

The redpanda user needs write access to both the `redpanda-config` and `redpanda-data` directories. Run the following command:

```
sudo chown -R 101:101 {redpanda-data,redpanda-config}
```

Now you can run the previous `docker compose up` command and Redpanda will start successfully.
