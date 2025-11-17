*Mumbai (ap-south-1) ↔ Hyderabad (ap-south-2) | Active-Passive DR Setup*  
*Tested & working as of November 2025 | Author: @aura_farmerr + Grok*

---

## Why MirrorMaker 2?
- Fully controlled replication (topics, offsets, configs)
- Works with IAM-authenticated MSK clusters
- Supports unidirectional (DR) or bidirectional (active-active)
- Free (just pay for EC2 + data transfer)

> Note: AWS now recommends **MSK Replicator** for managed experience, but MM2 gives you full control.

---

## Prerequisites
- AWS account with permissions for MSK, VPC, EC2
- AWS CLI v2
- Java 11+ (Amazon Corretto)
- Two regions: `ap-south-1` (Mumbai) & `ap-south-2` (Hyderabad)

---

## Step 1: Create Minimal MSK Clusters

### Cluster 1 – Primary (Mumbai)
| Setting                  | Value                                      |
|--------------------------|--------------------------------------------|
| Region                   | ap-south-1                                 |
| Cluster name             | `msk-primary-mumbai`                       |
| Kafka version            | 3.7.x (latest)                             |
| Broker type              | Provisioned                                |
| Broker instance          | `kafka.t3.small` or `kafka.m5.large`       |
| Brokers                  | 2 (1 per AZ)                               |
| Storage                  | 100 GB EBS per broker                      |
| Authentication           | IAM role-based                             |
| Encryption               | TLS in-transit + AWS KMS at-rest           |
| Security Group           | Allow 9098 from VPC CIDR + EC2 SG          |

### Cluster 2 – DR (Hyderabad)
Same as above but:
- Region: `ap-south-2`
- Cluster name: `msk-dr-hyderabad`

Copy both **bootstrap server strings** (IAM port 9098).

---

## Step 2: VPC Peering (Mandatory for Private Connectivity)

1. From Mumbai → Create peering connection → Select Hyderabad VPC
2. Accept in Hyderabad region
3. Add routes in both VPCs:
   - Mumbai route table: `10.1.0.0/16` → peering connection
   - Hyderabad route table: `10.0.0.0/16` → peering connection
4. Update MSK security groups to allow 9098 from peer VPC CIDR

---

## Step 3: Launch EC2 for MirrorMaker (in Mumbai)

| Setting            | Value                     |
|--------------------|---------------------------|
| AMI                | Amazon Linux 2023         |
| Type               | t3.micro or t3.small      |
| VPC/Subnet         | Mumbai private subnet     |
| IAM Role           | With `AmazonMSKFullAccess`|
| Security Group     | SSH from your IP, outbound all |

SSH in and run:

```bash
sudo dnf update -y
sudo dnf install -y java-11-amazon-corretto wget
cd ~
wget https://archive.apache.org/dist/kafka/3.7.0/kafka_2.13-3.7.0.tgz
tar -xzf kafka_2.13-3.7.0.tgz
ln -s kafka_2.13-3.7.0 kafka
mkdir msk-certs && cd msk-certs

# MSK IAM Auth JAR
wget https://github.com/aws/aws-msk-iam-auth/releases/download/v2.2.0/aws-msk-iam-auth-2.2.0-all.jar
mv aws-msk-iam-auth-2.2.0-all.jar aws-msk-iam-auth.jar

# Truststore (run locally or on EC2)
wget https://www.amazontrust.com/repository/AmazonRootCA1.pem
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file AmazonRootCA1.pem -storepass anjali -noprompt






4. create a file called `~/msk-certs/client.properties`

```
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
ssl.truststore.location=/home/ec2-user/msk-certs/kafka.client.truststore.jks
ssl.truststore.password=anjali
```

5. Create file called `~/mm2.properties`

```

# Cluster definitions
clusters=primary,dr

primary.bootstrap.servers=b-1.mskprimaryxxxx.ap-south-1.amazonaws.com:9098,b-2.mskprimaryxxxx.ap-south-1.amazonaws.com:9098
dr.bootstrap.servers=b-1.mskdrxxxx.ap-south-2.amazonaws.com:9098,b-2.mskdrxxxx.ap-south-2.amazonaws.com:9098

# Security (repeat for both)
primary.security.protocol=SASL_SSL
dr.security.protocol=SASL_SSL
primary.sasl.mechanism=AWS_MSK_IAM
dr.sasl.mechanism=AWS_MSK_IAM
primary.sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
dr.sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
primary.sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
dr.sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
primary.ssl.truststore.location=/home/ec2-user/msk-certs/kafka.client.truststore.jks
dr.ssl.truststore.location=/home/ec2-user/msk-certs/kafka.client.truststore.jks
primary.ssl.truststore.password=anjali
dr.ssl.truststore.password=anjali

# REPLICATION RULES (THIS IS THE KEY PART)
primary->dr.enabled=true
dr->primary.enabled=false           # Set true only for active-active

primary->dr.topics=.*
primary->dr.groups=.*
primary->dr.topics.rename=primary.   # topics appear as primary.orders in DR

sync.topic.configs.enabled=true
primary->dr.emit.checkpoints.enabled=true
primary->dr.auto.create.topics.enable=true

tasks.max=4
replication.factor=2

```

6. Start mirrowmaker2

```
export CLASSPATH=/home/ec2-user/msk-certs/aws-msk-iam-auth.jar:/home/ec2-user/kafka/libs/*

nohup /home/ec2-user/kafka/bin/connect-mirror-maker.sh \
  /home/ec2-user/mm2.properties --cluster.id mm2-poc > mm2.log 2>&1 &
  
  
```

7.  Test replication

```
# Produce to primary
kafka/bin/kafka-console-producer.sh --topic orders --bootstrap-server <PRIMARY_BOOTSTRAP> --producer.config msk-certs/client.properties

# Consume from DR (note prefix!)
kafka/bin/kafka-console-consumer.sh --topic primary.orders --from-beginning --bootstrap-server <DR_BOOTSTRAP> --consumer.config msk-certs/client.properties

```

8.  Additional steps:

```
export CLASSPATH=/home/ec2-user/msk-certs/aws-msk-iam-auth.jar:/home/ec2-user/kafka/libs/* &&
/home/ec2-user/kafka/bin/kafka-console-producer.sh \
  --topic orders \
  --bootstrap-server b-1.mskprimaryxxxx.ap-south-1.amazonaws.com:9098,b-2.mskprimaryxxxx.ap-south-1.amazonaws.com:9098 \
  --producer.config /home/ec2-user/msk-certs/client.properties


## use this in DR:

export CLASSPATH=/home/ec2-user/msk-certs/aws-msk-iam-auth.jar:/home/ec2-user/kafka/libs/* &&
/home/ec2-user/kafka/bin/kafka-console-consumer.sh \
  --topic primary.orders \
  --from-beginning \
  --bootstrap-server b-1.mskdrxxxx.ap-south-2.amazonaws.com:9098,b-2.mskdrxxxx.ap-south-2.amazonaws.com:9098 \
  --consumer.config /home/ec2-user/msk-certs/client.properties

```
