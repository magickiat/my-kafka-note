# Setup Kafka with SSL บน Local

- Download Kafka 2.6.0 จาก http://kafka.apache.org/
- แตก zip
- ปรับ config log dirs ของ kafka และ zookeeper ไปที่อื่นที่ไม่ใช่ /tmp เพราะจะมีปัญหากับ os สาย linux

## Enable SSL

การเปิด SSL เราจะต้องมี Key ที่ Signed ด้วย CA ก่อน

เราจะใช้วิธีสร้าง CA key ขึ้นมาก่อน จากนั้นจึงสร้าง Key ของ Kafka

แล้วทำการ Sign public key ของ Kafka ด้วย CA key

หมายเหตุ: CN name จะต้องเป็นชื่อเครื่องเท่านั้น ไม่อย่างนั้นจะเกิดเหตุการณ์ที่ Kafka ตรวจสอบ Hostname ไม่ผ่าน หากต้องการเปลี่ยน จะต้องไปปิด Host Name Verification ทั้งฝั่ง Client และ Kafka โดยดูจาก Kafka Document ในหัวข้อ **Host Name Verification**

### Gen SSL Key

1. สร้างไฟล์ชื่อ `create-key.sh`
2. สร้าง CA key ด้วยคำสั่ง 

```shell
#!/bin/bash

echo "##################################"
echo "            Generate KEY"
echo "##################################"
# Create CA
echo "# clear old key"
rm -rf gen-key

mkdir gen-key

echo "##### 1) GEN CA KEY #####"
VALIDITY=3650
CA_PWD="2vyf]VV8{gqq8GGd"
CA_KEY="gen-key/ca.key"
CA_CERT="gen-key/ca.crt"
CA_ALIAS="kafka-ca"

openssl req \
	-new \
	-newkey rsa:4096 \
	-days $VALIDITY \
	-x509 -subj "/CN=Kafka-Security-CA" \
	-keyout $CA_KEY \
	-out $CA_CERT \
	-nodes
```

2. สร้าง Key ของ Kafka และทำการ Signed ด้วย CA จากข้อ 1

```shell
echo "##### 2) GEN KAFKA KEY #####"
KAFKA_PWD="9[89Ej~XH_YC9%JL"
KAFKA_KEYSTORE="gen-key/server.keystore.jks"
KAFKA_TRUSTSTORE="gen-key/server.truststore.jks"
KAFKA_CSR="gen-key/server-cert-file"
KAFKA_ALIAS="kafka-localhost"
KAFKA_SIGNED="gen-key/server-cert-file"

echo "# gen keystore"
#keytool -genkey -keyalg RSA -keysize 4096 -alias $KAFKA_ALIAS -keystore $KAFKA_KEYSTORE -validity $VALIDITY -storepass $KAFKA_PWD -keypass $KAFKA_PWD -dname "CN=kafka-localhost" -storetype pkcs12
keytool -genkey \
		-keyalg RSA \
		-keysize 4096 \
		-alias $KAFKA_ALIAS \
		-keystore $KAFKA_KEYSTORE \
		-validity $VALIDITY \
		-storepass $KAFKA_PWD \
		-keypass $KAFKA_PWD \
		-dname "CN=localhost" 
		-storetype pkcs12

echo "# gen cert request"
keytool -keystore $KAFKA_KEYSTORE \
		-certreq \
		-alias $KAFKA_ALIAS \
		-file $KAFKA_CSR \
		-storepass $KAFKA_PWD \
		-keypass $KAFKA_PWD

echo "# Sign"
openssl x509 \
		-req \
		-CA $CA_CERT \
		-CAkey $CA_KEY \
		-in $KAFKA_CSR \
		-out $KAFKA_SIGNED \
		-days $VALIDITY \
		-CAcreateserial \
		-passin pass:$KAFKA_PWD

echo "# create truststore"
keytool -keystore $KAFKA_TRUSTSTORE \
		-alias $CA_ALIAS \
		-import \
		-file $CA_CERT \
		-storepass $KAFKA_PWD \
		-noprompt

echo "# import signed cert"
keytool -keystore $KAFKA_KEYSTORE \
		-alias $CA_ALIAS \
		-import \
		-file $CA_CERT \
		-storepass $KAFKA_PWD \
		-noprompt
keytool -keystore $KAFKA_KEYSTORE \
		-alias $KAFKA_ALIAS \
		-import \
		-file $KAFKA_SIGNED \
		-storepass $KAFKA_PWD \
		-noprompt
```

3. สร้าง Truststore ฝั่ง Client

```shell
echo "##### 3) Gen Client Key #####"

CLIENT_ALIAS="my-local-dev"
CLIENT_PWD=":*Jh?ZNgB}2?vw}["
CLIENT_TRUSTSTORE="gen-key/client.truststore.jks"
CLIENT_KEYSTORE="gen-key/client.keystore.jks"
CLIENT_CSR="gen-key/client-cert-request"
CLIENT_SIGNED="gen-key/client-cert-file"

echo "# gen truststore"
keytool -keystore $CLIENT_TRUSTSTORE \
		-alias $CA_ALIAS \
		-import \
		-file $CA_CERT \
		-storepass $CLIENT_PWD \
		-noprompt
```

4. สั่ง `chmod +x create-key.sh` เพื่อทำให้รันไฟล์ได้
5. จากนั้นรันคำสั่ง `./create-key.sh` เพื่อเริ่ม gen key

```
##################################
            Generate KEY
##################################
# clear old key
##### 1) GEN CA KEY #####
Generating a 4096 bit RSA private key
............................................................................................................................................................................++
......++
writing new private key to 'gen-key/ca.key'
-----
##### 2) GEN KAFKA KEY #####
# gen keystore
./create-key-2.sh: line 47: -storetype: command not found
# gen cert request
# Sign
Signature ok
subject=/CN=localhost
Getting CA Private Key
# create truststore
Certificate was added to keystore
# import signed cert
Certificate was added to keystore
Certificate reply was installed in keystore
##### 3) Gen Client Key #####
# gen truststore
Certificate was added to keystore
```

### Set kafka to use SSL

1. ไปที่โฟลเดอร์ kafka/config จากนั้นทำการ copy ไฟล์ `server.properties` แล้วตั้งชื่อใหม่เป็น `server-ssl.properties`
2. แก้ไข/เพิ่ม config ใน `server-ssl.properties` ดังนี้ (อย่าลืมเปลี่ยน path และ password หากมีการเปลี่ยนค่า)

```properties
listeners=SSL://:9092
advertised.listeners=SSL://localhost:9092

##### Security Config #####
security.inter.broker.protocol=SSL
ssl.keystore.location=/Applications/Development/kafka/keys/gen-key/server.keystore.jks
ssl.keystore.password=9[89Ej~XH_YC9%JL
ssl.key.password=9[89Ej~XH_YC9%JL
ssl.truststore.location=/Applications/Development/kafka/keys/gen-key/server.truststore.jks
ssl.truststore.password=9[89Ej~XH_YC9%JL
```

### ทดสอบ Start Server

1. รันคำสั่ง `bin/zookeeper-server-start.sh config/zookeeper.properties`
2. รันคำสั่ง `bin/kafka-server-start.sh config/server-ssl.properties`
3. จะต้องไม่มี error
4. เปิด terminal ใหม่ แล้วเช็คว่า Kafka เปิดเป็นแบบ SSL หรือไม่ด้วยคำสั่ง ```openssl s_client -connect localhost:9092``` จะได้รายละเอียด SSL ประมาณนี้

```
CONNECTED(00000005)
depth=1 CN = Kafka-Security-CA
verify error:num=19:self signed certificate in certificate chain
verify return:0
---
Certificate chain
 0 s:/CN=localhost
   i:/CN=Kafka-Security-CA
 1 s:/CN=Kafka-Security-CA
   i:/CN=Kafka-Security-CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIErDCCApQCCQDbhRITrXyySDANBgkqhkiG9w0BAQUFADAcMRowGAYDVQQDDBFL
YWZrYS1TZWN1cml0eS1DQTAeFw0yMDEwMDMxNTQwMzNaFw0zMDEwMDExNTQwMzNa
MBQxEjAQBgNVBAMTCWxvY2FsaG9zdDCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCC
AgoCggIBAIGkrP7CwP5TYvzI7dHucDJwZf4HrKz+HyhilxDfBbb+goyq/7s7k6vh
h9Vy31IDxQJbVsHnzdbPGyhhPRA+bQ6e59cwTWyAYHxDdfGy9TB7/VlY903Y/POC
kViAISkk/OgcqDp9WpM+hr2rwTuPkkr/Et6PIIQxxi/FtTy4NvzrphcFK2TC2gO/
caMtDn+TRO6RDKy3OMd1jbk9sGIS+2cDAkUugKtL7sdVGTVWlZxtaHs9wf04rsV7
zMbJnu0zXz6bPbLieDgfuD60Tz428B+IhhQrSl8hxVSze2FzWgqbyMb2CLI8rVXQ
bkK6dao8PboCSVENaJ6nBKjxayno8uPfco4bjpxGVMBQHuFGWRO9FuA5Ar3BuMYn
5t9ITIkEoVMrAXe6+Os4ue3SOXJclBmwmXq6hk4B7ANzw6pyft7UEM5B7QJKFxqY
ASe2AFvRJ0ZzUv2zVTzWHwmi83foWegKsRnW2J19KFeQbeKjllFX0W32BmG1F7Rg
DHjPPIwuvLKY6a1HAzufn/WOb1MZ0vkC0xPHg8A6hc6J6h3p4OYfqz1F/cTQWjqo
80tSPu+cujmYytm+5K+zjhZw6ca7mf8FpMHumi4IVc7BylVEAkYsaVqTxlrDpUQt
b5RbXf8Ez8zsfaUwgDMwpzHs5mu+Y4wTUxuguOgBQcNUSsBzTorfAgMBAAEwDQYJ
KoZIhvcNAQEFBQADggIBALUq1a67lbB5guMIkuPO7H2RkTDLm656MUTSEMlxR0sG
BTHIKdC65DdM/TuNOpSmI0NvXPH2kjHNUV82b32IjwZYFrJdjReEnauy7Omt9shW
NriJBK9lCpas6o+Cf/8/Xs1qzqyNBgr7EmSHP1JeBx2sP21Ursa0nlX+moYruXVV
oePn4ZXmrXI/EgHDul3fo0UnsJhRD8JyFX3doNIQMZUxlqSApzcA/oUSj3QH1iyX
NPqwGvJUwjRnEMWNr32G3aWsUO3MtcaF6I+/JGpVq12llCGVEEQcRip76ENItcfz
s3TuaugN1ZFWwAfj8JuSFssLMdxGaYloF8bgyiHnsKEHm10U0N/bcd7p51GmGxsT
ILIIt1QdmiwTh0MtNH9md1iVAQ5oix9/gjZKag20fKtafuupnC0B7RvfTU+MbPeW
WobcB7PSDXQyXuxdlNg+PJon3eUTmd+bBY/guakwOa3SJokDL/kN6FwCRFD3yg4X
B9O3BmQN4KObNT5UcIX/2ui8bRzqMhkIw0t8qbfQ9kD+/bxyUI+WxGIXVVLwqcQt
WaPjAehvf8GoxQMtUl/vnT+uSHpoUogZUpZk6GinHjjmG95iEvdaGFGsC/apwt1K
nI5k9fAJCtA8RyMitpxvo9xyI3JStALXVRde2VfuJLXz0ylm9zBPtrE3IkSXPyhn
-----END CERTIFICATE-----
subject=/CN=localhost
issuer=/CN=Kafka-Security-CA
---
No client certificate CA names sent
Server Temp Key: ECDH, P-256, 256 bits
---
SSL handshake has read 3151 bytes and written 322 bytes
---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 4096 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: 252EA7265A98D3AE704C99A988DB4EB0120AC2D10F29740455F4365CE5FDDB9F
    Session-ID-ctx: 
    Master-Key: B8F5017D9B4E0D6C480853043F6D731A78487D7684177FFB49EDEE45E892DB79FE60906BB18B3BADCC8F7F090F506518
    Start Time: 1601740940
    Timeout   : 7200 (sec)
    Verify return code: 19 (self signed certificate in certificate chain)
---

```

5. กด control + c เพื่อออกจาก openssl

### Spring Boot Kafka client

ทดสอบ Kafka Client โดยสร้าง Spring Boot แล้วเลือก dependencies เป็น Spring Boot Kafka ตัวเดียว

1. แก้ไข Main Class เพื่อให้ publish message แบบ Command Line Runner ดังนี้

```java
package cc.magickiat.kafka.demo;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.kafka.core.KafkaTemplate;

@SpringBootApplication
public class ProducerSslDemoApplication implements CommandLineRunner {
	
	@Autowired
	private KafkaTemplate<String, String> template;

	public static void main(String[] args) {
		SpringApplication.run(ProducerSslDemoApplication.class, args);
	}

	@Override
	public void run(String... args) throws Exception {
		String topic = "hellossl";
		String data = "Hello World Kafka SSL!!!";
		
		template.send(topic, data);
	}

}
```

2. เพิ่ม config ใน application.properties เพื่อให้ใช้งาน SSL ดังนี้ (อย่าลืมเปลี่ยน path และ password หากมีการเปลี่ยนค่า)

```properties
spring.kafka.ssl.trust-store-location=file:///Applications/Development/kafka/keys/gen-key/client.truststore.jks
spring.kafka.ssl.trust-store-type=pkcs12
spring.kafka.ssl.trust-store-password=:*Jh?ZNgB}2?vw}[

spring.kafka.security.protocol=SSL
```

3. ทำการรัน Main Class ก็จะสามารถ publish message ผ่าน SSL ได้แล้ว

```

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.4.RELEASE)

2020-10-03 22:58:20.027  INFO 2888 --- [           main] c.m.k.demo.ProducerSslDemoApplication    : Starting ProducerSslDemoApplication on MagicBookPro with PID 2888 (/Users/magicalcyber/Documents/workspaces/kafka-training/producer-ssl-demo/target/classes started by magicalcyber in /Users/magicalcyber/Documents/workspaces/kafka-training/producer-ssl-demo)
2020-10-03 22:58:20.029  INFO 2888 --- [           main] c.m.k.demo.ProducerSslDemoApplication    : No active profile set, falling back to default profiles: default
2020-10-03 22:58:20.476  INFO 2888 --- [           main] c.m.k.demo.ProducerSslDemoApplication    : Started ProducerSslDemoApplication in 0.635 seconds (JVM running for 1.107)
2020-10-03 22:58:20.488  INFO 2888 --- [           main] o.a.k.clients.producer.ProducerConfig    : ProducerConfig values: 
	acks = 1
	batch.size = 16384
	bootstrap.servers = [localhost:9092]
	buffer.memory = 33554432
	client.dns.lookup = default
	client.id = producer-1
	compression.type = none
	connections.max.idle.ms = 540000
	delivery.timeout.ms = 120000
	enable.idempotence = false
	interceptor.classes = []
	key.serializer = class org.apache.kafka.common.serialization.StringSerializer
	linger.ms = 0
	max.block.ms = 60000
	max.in.flight.requests.per.connection = 5
	max.request.size = 1048576
	metadata.max.age.ms = 300000
	metadata.max.idle.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partitioner.class = class org.apache.kafka.clients.producer.internals.DefaultPartitioner
	receive.buffer.bytes = 32768
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 30000
	retries = 2147483647
	retry.backoff.ms = 100
	sasl.client.callback.handler.class = null
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.login.callback.handler.class = null
	sasl.login.class = null
	sasl.login.refresh.buffer.seconds = 300
	sasl.login.refresh.min.period.seconds = 60
	sasl.login.refresh.window.factor = 0.8
	sasl.login.refresh.window.jitter = 0.05
	sasl.mechanism = GSSAPI
	security.protocol = SSL
	security.providers = null
	send.buffer.bytes = 131072
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2]
	ssl.endpoint.identification.algorithm = https
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLSv1.2
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = /Applications/Development/kafka/keys/gen-key/client.truststore.jks
	ssl.truststore.password = [hidden]
	ssl.truststore.type = pkcs12
	transaction.timeout.ms = 60000
	transactional.id = null
	value.serializer = class org.apache.kafka.common.serialization.StringSerializer

2020-10-03 22:58:20.706  INFO 2888 --- [           main] o.a.kafka.common.utils.AppInfoParser     : Kafka version: 2.5.1
2020-10-03 22:58:20.707  INFO 2888 --- [           main] o.a.kafka.common.utils.AppInfoParser     : Kafka commitId: 0efa8fb0f4c73d92
2020-10-03 22:58:20.707  INFO 2888 --- [           main] o.a.kafka.common.utils.AppInfoParser     : Kafka startTimeMs: 1601740700705
2020-10-03 22:58:21.052  WARN 2888 --- [ad | producer-1] org.apache.kafka.clients.NetworkClient   : [Producer clientId=producer-1] Error while fetching metadata with correlation id 1 : {hellossl=LEADER_NOT_AVAILABLE}
2020-10-03 22:58:21.054  INFO 2888 --- [ad | producer-1] org.apache.kafka.clients.Metadata        : [Producer clientId=producer-1] Cluster ID: 2y3Y0yyRToi94W24ZvB-iw
2020-10-03 22:58:21.200  WARN 2888 --- [ad | producer-1] org.apache.kafka.clients.NetworkClient   : [Producer clientId=producer-1] Error while fetching metadata with correlation id 3 : {hellossl=LEADER_NOT_AVAILABLE}
2020-10-03 22:58:21.321  INFO 2888 --- [extShutdownHook] o.a.k.clients.producer.KafkaProducer     : [Producer clientId=producer-1] Closing the Kafka producer with timeoutMillis = 30000 ms.

```

## SSL Client Authentication

จากอย่างที่แล้ว Client สามารถ publish message ได้เพียงแค่ทำ trust cert เท่านั้น หากต้องการบังคับให้ Client จะต้อง Signed ด้วย CA เหมือนกันเพื่อยืนยันตัวตนของ Client เราสามารถทำได้โดยการเพิ่ม config ใน `server-ssl.properties` ดังนี้

```properties
ssl.client.auth=required
```

จากนั้นทำการ restart kafka แล้วรัน Client อีกครั้งหนึ่ง จะพบ error ดังนี้

```
Caused by: org.apache.kafka.common.errors.SslAuthenticationException: SSL handshake failed
Caused by: javax.net.ssl.SSLProtocolException: Unexpected handshake message: server_hello
```

ดังนั้น เราจะต้องทำการสร้าง private key แล้วทำการ signed cert ด้วย CA ดังนี้

1. เพิ่มคำสั่งเหล่านี้ต่อจากไฟล์ `create-key.sh`

```shell
echo "# gen keystore"
keytool -genkey -keyalg RSA \
		-keysize 4096 \
		-alias $CLIENT_ALIAS \
		-keystore $CLIENT_KEYSTORE \
		-validity $VALIDITY \
		-storepass $CLIENT_PWD \
		-keypass $CLIENT_PWD \
		-dname "CN=localhost" \
		-storetype pkcs12

echo "# gen cert request"
keytool -keystore $CLIENT_KEYSTORE \
		-certreq \
		-alias $CLIENT_ALIAS \
		-file $CLIENT_CSR \
		-storepass $CLIENT_PWD \
		-keypass $CLIENT_PWD

echo "# CA sign cert request"
openssl x509 \
		-req \
		-CA $CA_CERT \
		-CAkey $CA_KEY \
		-in $CLIENT_CSR \
		-out $CLIENT_SIGNED \
		-days $VALIDITY \
		-CAcreateserial \
		-passin pass:$CLIENT_PWD

echo "# import signed cert"
keytool -keystore $CLIENT_KEYSTORE \
		-alias $CA_ALIAS \
		-import \
		-file $CA_CERT \
		-storepass $CLIENT_PWD \
		-noprompt
keytool -keystore $CLIENT_KEYSTORE \
		-alias $CLIENT_ALIAS \
		-import \
		-file $CLIENT_SIGNED \
		-storepass $CLIENT_PWD \
		-noprompt
```

2. รันซ้ำเพื่อ gen key และ signed cert
3. Restart Kafka (ที่ต้อง restart เนื่องจากเรารัน gen key ทั้งหมดใหม่ทั้งหมด)
4. เพิ่ม config truststore ฝั่ง client ในไฟล์ application.properties

```properties
spring.kafka.ssl.key-store-location=file:///Applications/Development/kafka/keys/gen-key/client.keystore.jks
spring.kafka.ssl.key-store-type=pkcs12
spring.kafka.ssl.key-store-password=:*Jh?ZNgB}2?vw}[
```

5. Run client ใหม่อีกครั้งก็จะ publish message ได้ปกติ

## Kafka Authentication with SASL

ตัวอย่างที่แล้วเป็นการเปิดใช้งานและทำ Authentication ที่ระดับ SSL คราวนี้เราจะมาทำ Authentication ที่ระดับ Kafka 

Kafka ใช้ Java Authentication and Authorization Service (JAAS) ในการทำ Config สำหรับกระบวนการ Login

### JAAS Config

JAAS config สำหรับ Kafka

- **KafkaServer** สำหรับ Broker แต่ละตัวคุยกัน และไว้คุยกับ Client

- **Client** สำหรับเชื่อมต่อกับ Zookeeper

JAAS config สำหรับ Client

- **KafkaClient** สำหรับคุยกับ Broker

### SASL Config

Kafka รองรับ SASL mechanisms ดังนี้

- [GSSAPI](http://kafka.apache.org/documentation/#security_sasl_kerberos) (Kerberos)
- [PLAIN](http://kafka.apache.org/documentation/#security_sasl_plain)
- [SCRAM-SHA-256](http://kafka.apache.org/documentation/#security_sasl_scram)
- [SCRAM-SHA-512](http://kafka.apache.org/documentation/#security_sasl_scram)
- [OAUTHBEARER](http://kafka.apache.org/documentation/#security_sasl_oauthbearer)

ที่จะเลือกใช้ตอนนี้คือ SCRAM-SHA-512 เนื่องจากต้องการแบบ Username/Password อยู่แต่ก็ไม่ต้องการให้เป็นแบบ PLAIN

### SASL/SCRAM Authentication

1. สร้าง User แบบ SCRAM บน Zookeeper ด้วยคำสั่ง

```shell
bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password=magickiatsecret]' --entity-type users --entity-name magickiat

bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password=adminsecret]' --entity-type users --entity-name admin
```

โดย `admin` จะใช้สำหรับ Kafka ส่วน `magickiat` จะใช้สำหรับ Client

หากสร้างสำเร็จจะขึ้นข้อความดังนี้

```
Warning: --zookeeper is deprecated and will be removed in a future version of Kafka.
Use --bootstrap-server instead to specify a broker to connect to.
Completed updating config for entity: user-principal 'magickiat'.

Warning: --zookeeper is deprecated and will be removed in a future version of Kafka.
Use --bootstrap-server instead to specify a broker to connect to.
Completed updating config for entity: user-principal 'admin'.
```

สำหรับการดู User ใน Zookeeper ให้ใช้คำสั่ง

```shell
bin/kafka-configs.sh --zookeeper localhost:2181 --describe --entity-type users --entity-name admin

bin/kafka-configs.sh --zookeeper localhost:2181 --describe --entity-type users --entity-name magickiat
```

จะได้หน้าตาดังนี้

```
Warning: --zookeeper is deprecated and will be removed in a future version of Kafka.
Use --bootstrap-server instead to specify a broker to connect to.
Configs for user-principal 'admin' are SCRAM-SHA-512=salt=MXRuNjZvcGc3b3M1d2U2bzVyNTc2czRvNQ==,stored_key=DTeL62W8fNiKRaDtkEAlcRxJPN+NgJdkQ2xLBbkuZCdsbg2jOpkhh0j59j2pYuof2LeEa+JbGcNHN7QWqLFTzQ==,server_key=ffiaFkg/UWL7M343JTi9GSvkym0beDqdM++HIyHOqpXJWCn/Xz3BCukoKDbU3etihpRGWX7xIu6TP0TtcpnCew==,iterations=4096

Warning: --zookeeper is deprecated and will be removed in a future version of Kafka.
Use --bootstrap-server instead to specify a broker to connect to.
Configs for user-principal 'magickiat' are SCRAM-SHA-512=salt=Mmo4eDllczVsOXE3aWQzMnF4NDA4YTZwcw==,stored_key=WVlxvAect0CP6CNa4/ngNBEMfIc5Z3rHJTG3W6FmlPvov8toZImdqWwJSQC3NUUJ5mK2DurgMYSmFmMGKrfcHw==,server_key=MWaHgCzTWU0lVnerx8FNiU1CkWQw1pdbrFyDBljW5v9+4BZScpJ1CVmemYjAaVI67tnKQ9cmKsl96jmTYhrY/g==,iterations=4096
```

2. สร้าง JAAS config สำหรับ Kafka โดยตั้งชื่อว่า `kafka_server_jaas.conf` เก็บไว้ในโฟลเดอร์ config

```
KafkaServer {
	org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="adminsecret";
};
```

3. แก้ไข server-ssl.properties ใหม่ดังนี้

```properties
listeners=SASL_SSL://:9092
advertised.listeners=SASL_SSL://localhost:9092

security.inter.broker.protocol=SASL_SSL

sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
```



4. Stop Kafka จากนั้นสร้างไฟล์ เพื่อใช้ในการรัน Kafka ขึ้นมา เพื่อทำการส่ง JAAS เข้าไปใน Kafka ดังนี้

```shell
#!/bin/bash
export KAFKA_OPTS=-Djava.security.auth.login.config=/Applications/Development/kafka/kafka_2.13-2.6.0/config/kafka_server_jaas.conf
bin/kafka-server-start.sh config/server-ssl.properties
```

Save ไว้ที่โฟลเดอร์ Kafka ชื่อ `run-kafka-ssl.sh`

5. รันคำสั่ง `chmod +x run-kafka-ssl.sh` เพื่อให้ script รันได้

6. จากนั้น start Kafka คำสั่ง `run-kafka-ssl.sh`

7. หากยิง client ตอนนี้ จะขึ้น error ว่า

```
2020-10-04 00:33:32.379  WARN 7521 --- [ad | producer-1] org.apache.kafka.clients.NetworkClient   : [Producer clientId=producer-1] Bootstrap broker localhost:9092 (id: -1 rack: null) disconnected
```

และฝั่ง Kafka จะขึ้น log ว่าให้ส่ง SASL เพื่อทำการ Authentication

```
[2020-10-04 00:33:32,951] INFO [SocketServer brokerId=0] Failed authentication with localhost/127.0.0.1 (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)
```

8. เพิ่ม/แก้ไข config JAAS ใน application.properties ได้เลย หรือจะแยกเป็นไฟล์ก็ได้ ในที่นี้จะใช้ในไฟล์ properties

```properties
spring.kafka.security.protocol=SASL_SSL

spring.kafka.bootstrap-servers=SASL_SSL://localhost:9092
spring.kafka.properties.sasl.mechanism=SCRAM-SHA-512
spring.kafka.properties.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="magickiat" \
        password="magickiatsecret";
```

9. จากนั้นลองรัน client ใหม่อีกครั้ง ก็จะสามารถ publish message ได้แล้ว

## Authorization

การกำหนดสิทธิของ User นั้นจะกำหนดและเก็บไว้ใน Zookeeper 

โดย Permission เราจะเรียกว่า ACLs 

1. ที่ server-ssl.properties ให้เพิ่ม config

```properties
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
allow.everyone.if.no.acl.found=false
```

2. จากนั้นทำการ restart kafka จะพบ error ไม่สามารถ start ได้เนื่องจาก user `admin` ไม่ได้รับอนุญาต

```
[2020-10-04 00:55:37,009] ERROR [KafkaApi-0] Error when handling request: clientId=0, correlationId=2, api=UPDATE_METADATA, version=6, body={controller_id=0,controller_epoch=7,broker_epoch=197,topic_states=[{topic_name=hellossl,partition_states=[{partition_index=0,controller_epoch=1,leader=0,leader_epoch=0,isr=[0],zk_version=0,replicas=[0],offline_replicas=[],_tagged_fields={}}],_tagged_fields={}}],live_brokers=[{id=0,endpoints=[{port=9092,host=localhost,listener=SASL_SSL,security_protocol=3,_tagged_fields={}}],rack=null,_tagged_fields={}}],_tagged_fields={}} (kafka.server.KafkaApis)
org.apache.kafka.common.errors.ClusterAuthorizationException: Request Request(processor=0, connectionId=127.0.0.1:9092-127.0.0.1:50020-0, session=Session(User:admin,localhost/127.0.0.1), listenerName=ListenerName(SASL_SSL), securityProtocol=SASL_SSL, buffer=null) is not authorized.
```

Log ที่เก็บเกี่ยวกับเรื่อง authorize จะอยู่ใน `logs/kafka-authorizer.log`

กรณีของ User admin ที่ Kafka ใช้นั้นไม่มีสิทธิ แต่ทว่า ตัว Kafka ไม่ได้เป็น User สำหรับใช้ publish/subscribe ข้อมูล จึงต้องสร้างเป็น Super User เพื่อไม่ต้องขอสิทธิในการทำงาน วิธีการทำคือไปเพิ่ม config ใน `server-ssl.properties` ดังนี้

```properties
super.users=User:admin
```

จากนั้นทำการ restart Kafka ก็จะสามารถใช้งานได้ โดยใน log จะปรากฏข้อความว่า 

```
[2020-10-04 01:01:05,191] WARN SASL configuration failed: javax.security.auth.login.LoginException: No JAAS configuration section named 'Client' was found in specified JAAS configuration file: '/Applications/Development/kafka/kafka_2.13-2.6.0/config/kafka_server_jaas.conf'. Will continue connection to Zookeeper server without SASL authentication, if Zookeeper server allows it. (org.apache.zookeeper.ClientCnxn)
```

3. เพื่อให้เป็นการรันที่ถูกวิธี เราจึงต้องไปเพิ่ม JAAS ฝั่ง Zookeeper ให้ทำการ Authen ก่อนการเข้าใช้งาน วิธีการรคือไปแก้ไฟล ​kafka_server_jaas.config เพิ่มส่วน Client เข้าไป

```
KafkaServer {
	org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="adminsecret";
};

Client {
   org.apache.zookeeper.server.auth.DigestLoginModule required
   username="zk-client"
   password="zk-client-secret";
};
```

และสร้างไฟล์ zk_server_jaas.conf เพื่อเก็บ User สำหรับการ Authentication

```
Server {
   org.apache.zookeeper.server.auth.DigestLoginModule required
   user_zk-client="zk-client-secret";
};
```

4. Stop Kafka และ Zookeeper
5. สร้าง shell สำหรับการรัน Zookeeper ที่ส่งไฟล์ JAAS เข้าไปเหมือนกับ Kafka ดังนี้

```shell
#!/bin/bash
export KAFKA_OPTS=-Djava.security.auth.login.config=/Applications/Development/kafka/kafka_2.13-2.6.0/config/zk_server_jaas.conf
bin/zookeeper-server-start.sh config/zookeeper.properties
```

6. ตั้งชื่อว่า `run-zk.sh` แล้วทำการรันคำสั่ง `chmod +x run-zk.sh` 
7. จากนั้นรัน Zookeeper ด้วยคำสั่ง `./run-zk.sh` ตามด้วย Kafka
8. Log ที่ฝั่ง Kafka ก็จะปรากฏ

```
[2020-10-04 01:21:45,478] INFO Client will use DIGEST-MD5 as SASL mechanism. (org.apache.zookeeper.client.ZooKeeperSaslClient)
```

ส่วนฝั่ง Zookeeper ก็จะปรากฏ log

```
[2020-10-04 01:21:43,155] INFO Successfully authenticated client: authenticationID=zk-client;  authorizationID=zk-client. (org.apache.zookeeper.server.auth.SaslServerCallbackHandler)
[2020-10-04 01:21:43,534] INFO Setting authorizedID: zk-client (org.apache.zookeeper.server.auth.SaslServerCallbackHandler)
[2020-10-04 01:21:43,535] INFO adding SASL authorization for authorizationID: zk-client (org.apache.zookeeper.server.ZooKeeperServer)
[2020-10-04 01:21:45,495] INFO Successfully authenticated client: authenticationID=zk-client;  authorizationID=zk-client. (org.apache.zookeeper.server.auth.SaslServerCallbackHandler)
[2020-10-04 01:21:45,495] INFO Setting authorizedID: zk-client (org.apache.zookeeper.server.auth.SaslServerCallbackHandler)
[2020-10-04 01:21:45,495] INFO adding SASL authorization for authorizationID: zk-client (org.apache.zookeeper.server.ZooKeeperServer)
```

9. ลองรัน Client เพื่อทดสอบส่งข้อความ จะพบ error ว่าไม่ได้รับอนุญาตให้เข้าใช้งาน

```
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [hellossl]
```

10. เพิ่ม ACLs เพื่อให้ user `magickiat` สามารถ Read, Write บน Topic `hellossl` และอนุญาตเฉพาะ host `localhost` เท่านั้น ด้วยคำสั่ง

```shell
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:magickiat --allow-host 127.0.0.1 --operation Read --operation Write --topic hellossl
```

จะปรากฏ log 

```
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=hellossl, patternType=LITERAL)`: 
 	(principal=User:magickiat, host=127.0.0.1, operation=WRITE, permissionType=ALLOW)
	(principal=User:magickiat, host=127.0.0.1, operation=READ, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=hellossl, patternType=LITERAL)`: 
 	(principal=User:magickiat, host=127.0.0.1, operation=WRITE, permissionType=ALLOW)
	(principal=User:magickiat, host=127.0.0.1, operation=READ, permissionType=ALLOW) 
```

11. เมื่อรัน Client อีกครั้ง ก็จะสามารถ publish message ได้ตามปกติ

### วิธีการดู Topic ACLs

1. รันคำสั่ง `bin/zookeeper-shell.sh localhost:2181` 
2. พิมพ์คำสั่ง `ls /kafka-acl/Topic` เพื่อดูว่ามี Topic ใหนที่เพิ่ม ACLs บ้าง
3. ดูค่าข้างในด้วยคำสั่ง `get /kafka-acl/Topic/hellossl` จะได้ค่าดังนี้

```
{"version":1,"acls":[{"principal":"User:magickiat","permissionType":"Allow","operation":"Write","host":"127.0.0.1"},{"principal":"User:magickiat","permissionType":"Allow","operation":"Read","host":"127.0.0.1"}]}
```

### Create ROOT user

ตอนนี้หากใช้คำสั่งสร้าง topic 

```shell
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic newtopic1
```

จะขึ้น error ดังนี้

```
Error while executing topic command : Call(callName=createTopics, deadlineMs=1601751816141, tries=1, nextAllowedTryMs=-9223372036854775709) timed out at 9223372036854775807 after 1 attempt(s)

[2020-10-04 02:02:36,323] ERROR org.apache.kafka.common.errors.TimeoutException: Call(callName=createTopics, deadlineMs=1601751816141, tries=1, nextAllowedTryMs=-9223372036854775709) timed out at 9223372036854775807 after 1 attempt(s)

Caused by: org.apache.kafka.common.errors.TimeoutException: The AdminClient thread has exited.
```

ฝั่ง Kafka จะขึ้น log ว่า 

```
[2020-10-04 02:08:52,251] INFO [SocketServer brokerId=0] Failed authentication with localhost/127.0.0.1 (SSL handshake failed) (org.apache.kafka.common.network.Selector)
```

เนื่องจากว่าเราเปิดใช้งาน SSL ดังนั้นทุกคำสั่งที่รันผ่าน bootstrap-server จะต้องมี SSL ซึ่งในกรณีนี้เราจำเป็นต้องมี SSL Key และ User แบบ SCRAM ด้วย ดังนั้นเราจึงต้อง Gen Key และสร้าง User ขึ้นมา

1. สร้าง user `root-kafka` 

   ```shell
   bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password=rootsecret]' --entity-type users --entity-name root-kafka
   ```

2. เพิ่ม step การสร้าง key ดังนี้

   ```shell
   echo "##### 4) Gen Super User Key #####"
   
   ROOT_ALIAS="super-user-kafka"
   ROOT_PWD="z3e5hKB]h&m3X&>5"
   ROOT_TRUSTSTORE="gen-key/root.truststore.jks"
   ROOT_KEYSTORE="gen-key/root.keystore.jks"
   ROOT_CSR="gen-key/root-cert-request"
   ROOT_SIGNED="gen-key/root-cert-file"
   
   echo "# gen truststore"
   keytool -keystore $ROOT_TRUSTSTORE \
   		-alias $CA_ALIAS \
   		-import \
   		-file $CA_CERT \
   		-storepass $ROOT_PWD \
   		-noprompt
   
   
   echo "# gen keystore"
   keytool -genkey \
   		-keyalg RSA \
   		-keysize 4096 \
   		-alias $ROOT_ALIAS \
   		-keystore $ROOT_KEYSTORE \
   		-validity $VALIDITY \
   		-storepass $ROOT_PWD \
   		-keypass $ROOT_PWD \
   		-dname "CN=localhost" \
   		-storetype pkcs12
   
   echo "# gen cert request"
   keytool -keystore $ROOT_KEYSTORE \
   		-certreq \
   		-alias $ROOT_ALIAS \
   		-file $ROOT_CSR \
   		-storepass $ROOT_PWD \
   		-keypass $ROOT_PWD
   
   echo "# CA sign cert request"
   openssl x509 \
   		-req \
   		-CA $CA_CERT \
   		-CAkey $CA_KEY \
   		-in $ROOT_CSR \
   		-out $ROOT_SIGNED \
   		-days $VALIDITY \
   		-CAcreateserial \
   		-passin pass:$ROOT_PWD
   
   echo "# import signed cert"
   keytool -keystore $ROOT_KEYSTORE \
   		-alias $CA_ALIAS \
   		-import \
   		-file $CA_CERT \
   		-storepass $ROOT_PWD \
   		-noprompt
   
   keytool -keystore $ROOT_KEYSTORE \
   		-alias $ROOT_ALIAS \
   		-import \
   		-file $ROOT_SIGNED \
   		-storepass $ROOT_PWD \
   		-noprompt
   ```

3. เพิ่ม `root-kaka` เข้าไปใน `super.users` ในไฟล์ `server-ssl.properties` แล้ว restart Kafka

   ```properties
   super.users=User:admin;User:root-kafka
   ```

4. เพิ่มไฟล์ `config/admin-client.properties` แล้วเพิ่ม config ดังนี้ (ตัวอย่างนี้ไม่ได้แยกไฟล์ JAAS ออกมา เพื่อให้สามารถใช้คำสั่งด้วยบรรทัดเดียว ไม่ต้อง export KAFKA_OPTS)

   ```properties
     security.protocol=SASL_SSL
     sasl.mechanism=SCRAM-SHA-512
   
     ssl.keystore.location=/Applications/Development/kafka/keys/gen-key/root.keystore.jks
     ssl.keystore.password=z3e5hKB]h&m3X&>5
     ssl.key.password=z3e5hKB]h&m3X&>5
     ssl.truststore.location=/Applications/Development/kafka/keys/gen-key/root.truststore.jks
     ssl.truststore.password=z3e5hKB]h&m3X&>5
   
     sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
             username="root-kafka" \
             password="rootsecret";
   ```

5. ทดสอบใช้คำสั่ง create-topic 

   ```shell
   bin/kafka-topics.sh --bootstrap-server localhost:9092 --command-config config/admin-client.properties --create --topic newtopic1
   ```

   ก็จะได้ผลลัพธ์ดังนี้

   ```
   Created topic newtopic1.
   ```

6. ลองเปลี่ยน user เป็น `magickiat` ที่ไม่มีสิทธิ super user 

   ```properties
   security.protocol=SASL_SSL
   sasl.mechanism=SCRAM-SHA-512
   
   ssl.keystore.location=/Applications/Development/kafka/keys/gen-key/root.keystore.jks
   ssl.keystore.password=z3e5hKB]h&m3X&>5
   ssl.key.password=z3e5hKB]h&m3X&>5
   ssl.truststore.location=/Applications/Development/kafka/keys/gen-key/root.truststore.jks
   ssl.truststore.password=z3e5hKB]h&m3X&>5
   
   # sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
   #         username="root-kafka" \
   #         password="rootsecret";
   
   sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
           username="magickiat" \
           password="magickiatsecret";
   ```

   ผลลัพธ์ที่ได้คือไม่สามารถสร้างได้

   ```
   Error while executing topic command : Authorization failed.
   [2020-10-04 15:01:21,569] ERROR org.apache.kafka.common.errors.TopicAuthorizationException: Authorization failed.
    (kafka.admin.TopicCommand$)
   ```

### Zookeeper Authentication

ลองไปดูว่า Topic ที่สร้างใหม่ด้วย super users นั้นมีใครสามารถเข้าใช้งานได้บ้าง จากตัวอย่างที่แล้วเราสามารถกำหนด ACLs ได้โดยรันคำสั่ง

```shell
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:magickiat --allow-host 127.0.0.1 --operation Read --operation Write --topic [topic_name]
```

แต่ทว่ามันเป็นการรันโดยต่อ Zookeeper ตรง ไม่ผ่าน Security ของ Kafka และในอนาคต ตัว Zookeeper จะถูกถอดออกไป ดังนั้นเราจึงต้องเปลี่ยนมาใช้คำสั่งที่เชื่อมต่อกับ Kafka แทน และวิธีการดู ACLs สามารถดูได้ด้วยคำสั่ง

```shell
bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config config/admin-client.properties --list --topic newtopic1
```

อย่าลืมลบ config ตอนเปลี่ยน user ด้วย จะได้ผลลัพธ์คือไม่มีค่าอะไรเลย หมายความว่าไม่มีใครสามารถทำอะไรกับ Topic นี้ได้ ดังนั้นเราจึงต้องตั้งค่า Kafka ให้ทำการเพิ่ม Default ACLs ให้ โดยไปเพิ่ม config นี้ในไฟล์ server-ssl.properties

```properties
zookeeper.set.acl=true
```

เมื่อ Restart Kafka ก็จะพบ Error

```
[2020-10-04 15:22:04,076] INFO [ZooKeeperClient Kafka server] Connected. (kafka.zookeeper.ZooKeeperClient)
[2020-10-04 15:22:04,135] ERROR Fatal error during KafkaServer startup. Prepare to shutdown (kafka.server.KafkaServer)
org.apache.zookeeper.KeeperException$InvalidACLException: KeeperErrorCode = InvalidACL for /brokers/ids
	at org.apache.zookeeper.KeeperException.create(KeeperException.java:128)
	at org.apache.zookeeper.KeeperException.create(KeeperException.java:54)
	at kafka.zookeeper.AsyncResponse.maybeThrow(ZooKeeperClient.scala:564)
	at kafka.zk.KafkaZkClient.createRecursive(KafkaZkClient.scala:1646)
	at kafka.zk.KafkaZkClient.makeSurePersistentPathExists(KafkaZkClient.scala:1568)
	at kafka.zk.KafkaZkClient.$anonfun$createTopLevelPaths$1(KafkaZkClient.scala:1560)
	at kafka.zk.KafkaZkClient.$anonfun$createTopLevelPaths$1$adapted(KafkaZkClient.scala:1560)
	at scala.collection.immutable.List.foreach(List.scala:333)
	at kafka.zk.KafkaZkClient.createTopLevelPaths(KafkaZkClient.scala:1560)
	at kafka.server.KafkaServer.initZkClient(KafkaServer.scala:445)
	at kafka.server.KafkaServer.startup(KafkaServer.scala:222)
	at kafka.server.KafkaServerStartable.startup(KafkaServerStartable.scala:44)
	at kafka.Kafka$.main(Kafka.scala:82)
	at kafka.Kafka.main(Kafka.scala)
```

ฝั่ง Zookeeper แจ้ง error ว่า

```
[2020-10-04 15:22:04,080] INFO Successfully authenticated client: authenticationID=zk-client;  authorizationID=zk-client. (org.apache.zookeeper.server.auth.SaslServerCallbackHandler)
[2020-10-04 15:22:04,080] INFO Setting authorizedID: zk-client (org.apache.zookeeper.server.auth.SaslServerCallbackHandler)
[2020-10-04 15:22:04,080] INFO adding SASL authorization for authorizationID: zk-client (org.apache.zookeeper.server.ZooKeeperServer)
[2020-10-04 15:22:04,121] ERROR Missing AuthenticationProvider for sasl (org.apache.zookeeper.server.PrepRequestProcessor)
```

เราต้องไปเพิ่ม config ที่ Zookeeper เพื่อเปิด SASL 

1. ทำการ Stop Zookeeper และ Kafka จากนั้น Gen Key สำหรับ Zookeeper หนึ่งตัว ดังนี้ แล้วก็รันเพื่อ Create Key ใหม่

   ```shell
   echo "##### 5) Gen Zookeeper Key #####"
   
   ZK_ALIAS="zookeeper"
   ZK_PWD="pmKTRXLcWtW7Nf~k"
   ZK_TRUSTSTORE="gen-key/zk.truststore.jks"
   ZK_KEYSTORE="gen-key/zk.keystore.jks"
   ZK_CSR="gen-key/zk-cert-request"
   ZK_SIGNED="gen-key/zk-cert-file"
   
   echo "# gen truststore"
   keytool -keystore $ZK_TRUSTSTORE \
   		-alias $CA_ALIAS \
   		-import \
   		-file $CA_CERT \
   		-storepass $ZK_PWD \
   		-noprompt
   
   
   echo "# gen keystore"
   keytool -genkey \
   		-keyalg RSA \
   		-keysize 4096 \
   		-alias $ZK_ALIAS \
   		-keystore $ZK_KEYSTORE \
   		-validity $VALIDITY \
   		-storepass $ZK_PWD \
   		-keypass $ZK_PWD \
   		-dname "CN=localhost" \
   		-storetype pkcs12
   
   echo "# gen cert request"
   keytool -keystore $ZK_KEYSTORE \
   		-certreq \
   		-alias $ZK_ALIAS \
   		-file $ZK_CSR \
   		-storepass $ZK_PWD \
   		-keypass $ZK_PWD
   
   echo "# CA sign cert request"
   openssl x509 \
   		-req \
   		-CA $CA_CERT \
   		-CAkey $CA_KEY \
   		-in $ZK_CSR \
   		-out $ZK_SIGNED \
   		-days $VALIDITY \
   		-CAcreateserial \
   		-passin pass:$ZK_PWD
   
   echo "# import signed cert"
   keytool -keystore $ZK_KEYSTORE \
   		-alias $CA_ALIAS \
   		-import \
   		-file $CA_CERT \
   		-storepass $ZK_PWD \
   		-noprompt
   
   keytool -keystore $ZK_KEYSTORE \
   		-alias $ZK_ALIAS \
   		-import \
   		-file $ZK_SIGNED \
   		-storepass $ZK_PWD \
   		-noprompt
   
   
   echo "##################################"
   echo "            Finished"
   echo "##################################"
   ```

2. copy `zookeeper.properties` มาเป็น `zookeeper-ssl.properties` และเพิ่ม config ดังนี้

   ```properties
   secureClientPort=2181
   
   ##### Security Config #####
   requireClientAuthScheme=sasl
   authProvider.sasl=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
   serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
   
   ssl.keyStore.location=//Applications/Development/kafka/keys/gen-key/zk.keystore.jks
   ssl.keyStore.password=pmKTRXLcWtW7Nf~k
   ssl.trustStore.location=/Applications/Development/kafka/keys/gen-key/zk.truststore.jks
   ssl.trustStore.password=pmKTRXLcWtW7Nf~k
   ```

   อย่าลืมเปลี่ยน `clientPort` เป็น `secureClientPort`

3. เพิ่ม config ใน server-ssl.properties ดังนี้

   ```properties
   zookeeper.set.acl=true
   zookeeper.ssl.client.enable=true
   zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
   # define key/trust stores to use TLS to ZooKeeper; ignored unless zookeeper.ssl.client.enable=true
   zookeeper.ssl.keystore.location=/Applications/Development/kafka/keys/gen-key/server.keystore.jks
   zookeeper.ssl.keystore.password=9[89Ej~XH_YC9%JL
   zookeeper.ssl.truststore.location=/Applications/Development/kafka/keys/gen-key/server.truststore.jks
   zookeeper.ssl.truststore.password=9[89Ej~XH_YC9%JL
   ```

   ระวัง `zookeeper.ssl` จะเป็น key ของ Kafka ไม่ใช่ของ Zookeeper

4. แก้ไฟล์ `run-zk.sh` ให้อ่านไฟล์ properties ใหม่

   ```shell
   #!/bin/bash
   export KAFKA_OPTS=-Djava.security.auth.login.config=/Applications/Development/kafka/kafka_2.13-2.6.0/config/zk_server_jaas.conf
   bin/zookeeper-server-start.sh config/zookeeper-ssl.properties
   ```

5. ทำการ Start Zookeerp แล้วตรวจสอบว่า Zookeeper เปิดใช้งาน SSL หรือยังโดยใช้คำสั่ง

   ```shell
   openssl s_client -connect localhost:2181
   ```

   จะปรากฏ Certificate 

   ```
   CONNECTED(00000005)
   depth=1 CN = Kafka-Security-CA
   verify error:num=19:self signed certificate in certificate chain
   verify return:0
   4703170156:error:1401E412:SSL routines:CONNECT_CR_FINISHED:sslv3 alert bad certificate:/AppleInternal/BuildRoot/Library/Caches/com.apple.xbs/Sources/libressl/libressl-47.140.1/libressl-2.8/ssl/ssl_pkt.c:1200:SSL alert number 42
   4703170156:error:1401E0E5:SSL routines:CONNECT_CR_FINISHED:ssl handshake failure:/AppleInternal/BuildRoot/Library/Caches/com.apple.xbs/Sources/libressl/libressl-47.140.1/libressl-2.8/ssl/ssl_pkt.c:585:
   ---
   Certificate chain
    0 s:/CN=localhost
      i:/CN=Kafka-Security-CA
    1 s:/CN=Kafka-Security-CA
      i:/CN=Kafka-Security-CA
   ---
   Server certificate
   -----BEGIN CERTIFICATE-----
   MIIErDCCApQCCQDAVV2CvUYsEjANBgkqhkiG9w0BAQUFADAcMRowGAYDVQQDDBFL
   YWZrYS1TZWN1cml0eS1DQTAeFw0yMDEwMDQwODI4MTNaFw0zMDEwMDIwODI4MTNa
   MBQxEjAQBgNVBAMTCWxvY2FsaG9zdDCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCC
   AgoCggIBALtxio8R8AnzUnK0DkcVkUM294Kw7RrvOtlUjbpY/fAlyzeqSa/hJrbs
   Rc5ii0m7WByQOcaLDC1Lg6DLXQXRAQUwR2bxraFkWi6DE6yrjCmZwyhdD3b76Hsq
   6DG35EHcBr0h4xU6eEqcy9a9lz0p+IBASA+x6xDMCNo043ZLlrCYGHrSpvx5NaPb
   8W7Q464yBaLyfxMOmH6iZeM0PviasU2cF+6JYJ8o+QtzBqRHu797FpFO2oN19bsN
   V2my4ARAqyi93IVlb5Sc19FDwvyuN7iPy26lhXAfVqTVizuuIPFTUaBUa3rC3Tij
   gFwoAZdbs9yoqHzK+RNazHhNkpgaC9x5/SkywKvS45w/7rcAPAyvIPdE7Pldm02B
   GNAjrdPzzkQ7S/SfzOJih6eur73ch3Vwsq3ZKK6BuGPiFvqBIz0A/K46UpQy6hT8
   r1jyhZc9V8z07/ozh2G78hlq90SvXrSt20HZ6JqArKg/OqyjRB1UHC8KDNsJ2iVL
   /o5B7SeHrc/onsHbWVk4rjdGsflztPsNKAsir/qBHNCQDGzPFV7hvW61ZwdvYnh8
   M5dbwegupDZ70iLHwIpyQ/+RAVVpd9SLrHxVev8vTUa1zTyQKbTMPg8lAEmiqzJk
   N4Pcb1L2kG0pDM5jyGlaveYFEmGvmxaNLS2FkpRBsV15VaIGCsutAgMBAAEwDQYJ
   KoZIhvcNAQEFBQADggIBAGuWPz0xkokmX6a1fed4v37sSXYDxxOoIuAqxesOfcei
   GHxm8L+wtPiEbys0HqLnuaDNn92F5dsrPgkBmcXFHJ7fLEdu+VKfvR6J22tKBjU9
   27/r51m0PcG1jbcV5KWuvb70C920pOwVi++NyQ5vx+L8MYr+CA9CBSIWtFQONHHL
   RQZYzEvC5Uoa4zkMeWFt/r5ONbXFormURtDsnaNFdt7DWp/lKJjZBhrVISMxVA0M
   MP8+euM5oS5CWYldmbQfBRjKH74XNtICOXTF5kbUK0UEY2rwiBMecd/LQvvpmm7B
   b+vB+kn+YbKSptOPQVIgpeoX0NyhcBf0LACxwBsm9EsTt5Ba+O1Dj98XUpsqGLEG
   5fLjzlLcicdarkpJhZdgIjuXx2QV/mVu4TXEk4clIxRi7AgLw0QImoUY6u3ko/FZ
   yKm4K1zZjd7tbfMg368Y87YBoo0Ia5AGyzlArEgmZxkM3z+pCU0K7peteLG06QhW
   0QOnIqW60E3+IK/0nfHIKzEIWtnxSpUM+AYF7b2yrZ+PeGO++244f+Sm8CDtQEVL
   cQ7VIZiMddMQSjwCDp1MNzpP1bmy3MtxuwLWS+M3EnI0Papju2Td9ZRXdMovJcK0
   KNaBxgIHmBj4C2sKyaxgbwXkJSH+1U1Px8/vueDzmqTta4UawnxaBOym1cokFq6c
   -----END CERTIFICATE-----
   subject=/CN=localhost
   issuer=/CN=Kafka-Security-CA
   ---
   Acceptable client certificate CA names
   /CN=Kafka-Security-CA
   Server Temp Key: ECDH, P-256, 256 bits
   ---
   SSL handshake has read 3189 bytes and written 138 bytes
   ---
   New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES256-GCM-SHA384
   Server public key is 4096 bit
   Secure Renegotiation IS supported
   Compression: NONE
   Expansion: NONE
   No ALPN negotiated
   SSL-Session:
       Protocol  : TLSv1.2
       Cipher    : ECDHE-RSA-AES256-GCM-SHA384
       Session-ID: C0E2B499D6EEC957D2E9CB93A0AC9B79D8F47D8187EE63E485440E3AC7E763BA
       Session-ID-ctx: 
       Master-Key: 62DCEB084DDD05C3A2591667A33199BC5F126830D1DAA401E0F0D892AB7747A76683C698CB80AC69E5E3C2EFE8225232
       Start Time: 1601801643
       Timeout   : 7200 (sec)
       Verify return code: 19 (self signed certificate in certificate chain)
   ---
   ```

6. รันคำสั่งเพื่อทำการ migrate zookeeper เพื่อไม่ให้มีการเข้าถึงข้อมูลใน zookeeper โดยตรงได้

   ```shell
   bin/zookeeper-security-migration.sh --zookeeper.acl=secure --zookeeper.connect=localhost:2181 --zk-tls-config-file config/server-ssl.properties
   ```

   เราต้องใช้ server-ssl.properties เนื่องจากว่ามี config สำหรับเชื่อมต่อกับ Kafka แบบ SSL แล้ว

7. คราวนี้พอรันคำสั่งที่เชื่อมต่อกับ Zookeeper โดยตรงก็จะไม่สามารถใช้งานได้แล้ว เช่น

   ```shell
   bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:magickiat --allow-host 127.0.0.1 --operation Read --operation Write --topic hellossl
   ```

   ก็จะไม่มีผลลัพธ์ออกมา แต่ว่า log ฝั่ง Zookeeper จะแจ้งประมาณว่า

   ```
   Caused by: io.netty.handler.ssl.NotSslRecordException: not an SSL/TLS record: 0000002d000000000000000000000000000046500000000000000000000000100000000000000000000000000000000000
   ```

### วิธีจัดการ ACLs เมื่อไม่สามารถเชื่อมต่อ Zookeeper โดยตรงได้

ก่อนหน้านี้เราเข้าไปดู ACLs ของ Topic ได้โดยผ่านการเชื่อมต่อกับ Zookeeper โดยตรง ผ่านคำสั่ง

```shell
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:magickiat --allow-host 127.0.0.1 --operation Read --operation Write --topic hellossl
```

แต่คราวนี้เมื่อเปิด SSL แล้วเราจะไม่สามารถทำได้ โดยฝั่ง Zookeeper จะขึ้น Error ประมาณนี้

```
Caused by: io.netty.handler.ssl.NotSslRecordException: not an SSL/TLS record: 0000002d000000000000000000000000000046500000000000000000000000100000000000000000000000000000000000
```

เราจะต้องทำผ่าน bootstrap server เท่านั้น

เมื่อกลับไปดู ACLs ของ Topic `newtopic1` ก็จะยังคงเหมือนเดิมอยู่คือยังไม่มี ACLs

```shell
bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config config/admin-client.properties --list --topic newtopic1
```

ส่วน topic `hellossl`

```shell
bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config config/admin-client.properties --list --topic hellossl
```

ก็จะมี 2 record

```
Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=hellossl, patternType=LITERAL)`: 
 	(principal=User:magickiat, host=127.0.0.1, operation=READ, permissionType=ALLOW)
	(principal=User:magickiat, host=127.0.0.1, operation=WRITE, permissionType=ALLOW) 
```

สำหรับวิธีการเพิ่ม ACLs ในตอนนี้สมมุติว่าต้องการให้ Client สามารถ publish message เข้าไปยัง topic `newoptic1` ได้ หากรันตอนนี้ ฝั่ง client จะขึ้น error ว่า

```
Caused by: org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [newtopic1]
```

เพิ่ม ACLs ด้วยคำสั่ง

```shell
bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config config/admin-client.properties --add --allow-principal User:magickiat --allow-host 127.0.0.1 --operation Read --operation Write --topic newtopic1
```

เพียงเท่านี้ Client ก็จะสามารถ publish message เข้า topic `newtopic1` ได้แล้ว



### Zookeeper super admin

กรณีที่เรามีปัญหากับระบบ ACLs และเราไม่สามารถทำอะไรได้แล้ว เราต้องพึ่ง super user ของ Zookeeper ที่จะไม่สนใจ ACLs เลย

วิธีการสร้าง

1. เปิด terminal ไปที่โฟลเดอร์ Kafka จากนั้นพิมพ์คำสั่ง

   ```shell
   export ZK_CLASSPATH=$(pwd)/libs/*
   
   java -cp $ZK_CLASSPATH \
   org.apache.zookeeper.server.auth.DigestAuthenticationProvider \
   super_admin:zksuperadmin
   ```

   super_admin:zksuperadmin คือ username:password ที่ต้องการ ผลลัพธ์ที่ได้จะได้ Digest password ออกมา

   ```
   super_admin:zksuperadmin->super_admin:i5I0aNnrEriLpEoxDnQ0PD3sdok=
   ```

   

2. แก้ไขไฟล์ `run-zk.sh` ดังนี้ แล้วทำการ Restart Zookeeper

   ```shell
   #!/bin/bash
   export KAFKA_OPTS="-Djava.security.auth.login.config=/Applications/Development/kafka/kafka_2.13-2.6.0/config/zk_server_jaas.conf -Dzookeeper.DigestAuthenticationProvider.superDigest=super_admin:i5I0aNnrEriLpEoxDnQ0PD3sdok="
   bin/zookeeper-server-start.sh config/zookeeper-ssl.properties
   ```

3. รันคำสั่งเพื่อเข้า zookeeper sheel แบบ ssl

   ```shell
   bin/zookeeper-shell.sh localhost:2181 -zk-tls-config-file config/server-ssl.properties
   ```

   หากไม่มีปัญหาอะไร ลองพิมพ์คำสั่ง `ls /` เพื่อทดสอบว่ายังใช้งานได้อยู่

   ```
   Connecting to localhost:2181
   Welcome to ZooKeeper!
   JLine support is disabled
   
   WATCHER::
   
   WatchedEvent state:SyncConnected type:None path:null
   ls /
   [admin, brokers, cluster, config, consumers, controller_epoch, delegation_token, isr_change_notification, kafka-acl, kafka-acl-changes, kafka-acl-extended, kafka-acl-extended-changes, latest_producer_id_block, log_dir_event_notification, zookeeper]
   ```

4. ลองสร้าง node ใหม่ด้วยคำสั่ง

   ```
   create /zk_test
   ```

   จากนั้นลองดู ACLs ว่ามีอะไรบ้างด้วยคำสั่ง

   ```
   getAcl /zk_test
   'world,'anyone
   : cdrwa
   ```

   `'world,'anyone` ก็คือใครก็ได้ `cdrwa` คือ 

   - Create
   - Delete
   - Read
   - Write
   - Access

   ก็คือใครก็ได้จะทำอะไรก็ได้

5. ลอง setAcl เพื่อยกเลิกสิทธิด้วยคำสั่ง

   ```
   setAcl /zk_test world:anyone:
   ```

   จากนั้นลอง getAcl อีกครั้ง จะพบว่าเราไม่มีสิทธิ Access แล้ว

   ```
   getAcl /zk_test
   Authentication is not valid : /zk_test
   ```

6. เปลี่ยนตัวเองเป็น super admin ด้วยคำสั่ง

   ```
   addauth digest super_admin:zksuperadmin
   ```

   ที่ฝั่ง zookeeper จะขึ้นว่า

   ```
   [2020-10-04 17:29:23,213] INFO auth success /0:0:0:0:0:0:0:1:50038 (org.apache.zookeeper.server.ZooKeeperServer)
   ```

   เมื่อรันคำสั่ง getAcl อีกครั้งก็จะได้ข้อมูลออกมา ไม่ติด ACLs แล้ว

   ```
   getAcl /zk_test
   'world,'anyone
   : 
   ```

   









