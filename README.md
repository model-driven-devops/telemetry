# Telemetry
Per customer request, this repo has docker files that will help ingest Cisco Model-Driven-Telemetry (MDT) into elasticsearch. Cisco MDT uses gRPC to send data at a given time interval or when a change occures. The ELK stack currently has limited support for natively ingesting gRPC. Logstash does have a generic protobuf plugin, but it is complicated to set up and poorly documented. The TIG stack has native support for MDT, but sometimes customers have an existing tool they would prefer to take advantage of. 

To overcome the lack of MDT support using the ELK stack, we will use the MDT plugin for Telegraf and send data to Kafka. Logstash will then grab the data from Kafka and send it to elasticsearch.

Question: Why not just use telegraf to send directly to Logstash?
Answer: Because the Elasticsearch output plugin for Telegraf only supports 5.x and 7.x. It also only supports HTTP and not HTTPS. 

Assumptions
- You have a Cisco device to test with.
- You have an Ubuntu VM with Docker Compose. I'm mocking this up in Cisco Modeling Labs, so I am seperating the ELK stack from Telegraf and Kafka using two different ubuntu images.
- The Cisco device can talk to the Ubuntu image.
- You are using this for testing (most security is turned off)

Notes
- GNMI Dial In does not seem to have support on virtual devices.

## Telegraf and Kafka

You can can clone this repo or copy and paste. It's only a few files, so whatever is easiest. 

Assuming you have cloned this repo, lets start by editing your docker-compose file. All you really need is to insert your host (external) IP where it says INSERT EXTERNAL IP.

```
    environment:
      # KRaft settings
      - KAFKA_CFG_NODE_ID=0  # Adjust the node ID as needed
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
      # Listeners
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://(INSERT EXTERNAL IP):9094
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_CFG_INTER_BROKER_LISTENER_NAME=PLAINTEXT
```

Telegraf is simple. You don't even need to change anything.
```
telegraf:
    container_name: telegraf
    image: telegraf:latest
    depends_on:
      - kafka
    ports:
      - "57000:57000"  # Expose the Telegraf metrics port (adjust as needed)
    volumes:
      - ./conf/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
```

Next, create your telegraf.conf file. It should be nested under the conf/telegraf directory.

```
# Cisco MDT Telemetry
[[inputs.cisco_telemetry_mdt]]
  service_address = ":57000"
  transport = "grpc"
  # List of gRPC subscription(s).
  # Each subscription is identified by a unique name.
  # For each subscription, specify the destination (Kafka topic) and encoding.

[[outputs.kafka]]
  # Kafka broker addresses (comma-separated list).
  brokers = ["kafka:9092"]

  # Kafka topic(s) to which Telegraf will send data.
  topic = "kafka_topic_1"
```

Thats pretty much it. Now spin up your containers.
```
docker compose up
```

## Send it something

Now that your containers are up and running, we need to send some telemetry from our router. Here is a basic config that sends CPU utilization every 5 seconds. Not very sexy, but its simple for testing.

```
netconf-yang

telemetry ietf subscription 1
 encoding encode-kvgpb
 filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
 source-address <IP ADDRESS OF LOCAL INTERFACE>
 stream yang-push
 update-policy periodic 500
 receiver ip address <IP ADDRESS OF COMPUTER HOSTING CONTAINERS> 57000 protocol grpc-tcp
```
Your source address should be the interface you are sending data from and your reiever IP address should be the host IP where your containers live. To verify your router can communicate with telegraf, you can run this command:

```
show telemetry ietf subscription 1 receiver
```
You should see "connected" in the output.

```
Telemetry subscription receivers detail:

  Subscription ID: 1
  Address: 192.133.186.217
  Port: 57000
  Protocol: grpc-tcp
  Profile: 
  Connection: 1140
  State: Connected
  Explanation:
```

Now lets verify Kafka is recieving the data. Log back into your container hose and run the following command:

```
docker exec -it  kafka /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic kafka_topic_1 --from-beginning
```
This will run the console consumer script from inside of the Kafka container and output the data that has been collected for your topic. You should see a series of messages displaying CPU utilization.

## ELK Stack

You can obviously run all these containers on a single host and merge the docker compose files, BUT because the ELK stack is so weird with memory and heaps and swap partitions and java stuff or whatever, I find it easier to just seperate it onto a second ubuntu image so when I do have to inevitably troubleshoot, I'm only focusing on the ELK stack.

This is a modified version of the docker compose file from elastics website and assumes you already figured out you didn't allocate enough RAM and re-built your ubuntu image a few times.

The .env file includes the only variables you really need to change, or you can just accept the defaults. All you really need to do is point the logstash.conf file to your kafka host.

```
input {
  kafka {
    bootstrap_servers => "(INSERT KAFKA HOST IP):9094"
    topics => ["kafka_topic_1"]
  }
}

filter {
  json {
    source => "message"
    target => "parsed_json"
  }
}

output {
  elasticsearch {
   index => "kafka-%{+YYYY.MM.dd}"
   hosts=> "${ELASTIC_HOSTS}"
   user=> "${ELASTIC_USER}"
   password=> "${ELASTIC_PASSWORD}"
   cacert=> "certs/ca/ca.crt"
 }
}
```

Now you just gotta give it a docker compose up and watch the magic happen.

```
docker compose up
```
You should be able to log into kibana and see all your cool data.

![Screenshot 2023-08-14 at 3 28 33 PM](https://github.com/model-driven-devops/telemetry/assets/65776483/dc95f9bc-6ebf-4ce9-bd30-df791ac6b5a5)
