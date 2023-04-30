## 1. Network IDS: Suricata

References:
- How To Install Suricata on CentOS 8 Stream: <https://www.digitalocean.com/community/tutorials/how-to-install-suricata-on-centos-8-stream>
- Understanding Suricata Signatures: <https://www.digitalocean.com/community/tutorials/understanding-suricata-signatures>

### 1.1. Install Suricata:

```sh
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
yum -y copr enable @oisf/suricata-latest
yum -y install suricata
```

### 1.2. Edit Suricata configuration:

Multiple interfaces: Read "Set the Interface" <https://forum.suricata.io/t/guide-getting-started-on-centos-8-and-centos-7/538>

```sh
cp /etc/suricata/suricata.yaml /etc/suricata/suricata.yaml.bak
sed -i 's/      community-id: false/\      community-id: true/' /etc/suricata/suricata.yaml
echo -e "detect-engine:\n  - rule-reload: true" >> /etc/suricata/suricata.yaml
sed -i 's/-i eth0/-i eth0 -i eth1/' /etc/sysconfig/suricata
systemctl daemon-reload
```

### 1.3. Test and enable+start Suricata service:

```sh
sudo -u suricata suricata-update
systemctl enable --now suricata
systemctl status suricata
tail /var/log/suricata/suricata.log
```

### 1.4. Test IDS:

```sh
curl http://testmynids.org/uid/index.html
curl -Lk http://testmyids.com
tail -f /var/log/suricata/fast.log
```

> Attempting to `curl https://testmyids.com` didn't log any events, perhaps because it's encrypted?

- More IDS tests: <https://github.com/3CORESec/testmynids.org>

## 2. SIEM: Elasticsearch + Kibana

References:
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html>
- <https://www.elastic.co/guide/en/kibana/current/rpm.html>
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/security-minimal-setup.html>

### 2.1. Setup Elastic repository

Import GPG key

> The signature is SHA1
>
> `update-crypto-policies --set DEFAULT:SHA1` is required on newer OS
>
> Otherwise, the following error will be encountered when importing the GPG key
>
> ```
> warning: Signature not supported. Hash algorithm SHA1 not available.
> error: https://artifacts.elastic.co/GPG-KEY-elasticsearch: key 1 import failed.
> ```

```sh
update-crypto-policies --set DEFAULT:SHA1
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

Configure Elastic repository

```sh
cat << EOF >> /etc/yum.repos.d/elasticsearch.repo
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md
EOF
```

### 2.2. Install and configure Elastic Stack

Install Elastic search and Kibana

```sh
yum -y install --enablerepo=elasticsearch elasticsearch kibana
```

Allow access to Elastic Stack on firewall:

```sh
firewall-cmd --permanent --add-service=elasticsearch
firewall-cmd --permanent --add-service=kibana
firewall-cmd --reload
```

<details><summary><h3>2.3. Elastic default certificates</h3></summary>

Elasticsearch after version 8.0 are installed with security enabled by default

Ref: <https://www.elastic.co/guide/en/elasticsearch/reference/8.7/configuring-stack-security.html#stack-security-certificates>

Default security settings in `/etc/elasticsearch/elasticsearch.yml`:

```yaml
# Enable security features
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
```

Certificates `http.p12` and `transport.p12` at `/etc/elasticsearch/certs` are automatically generated upon installation and are signed by `Elasticsearch security auto-configuration HTTP CA`

Auto generated certificates example:

```console
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore list
autoconfiguration.password_hash
keystore.seed
xpack.security.http.ssl.keystore.secure_password
xpack.security.transport.ssl.keystore.secure_password
xpack.security.transport.ssl.truststore.secure_password
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show autoconfiguration.password_hash
$2a$10$xs1YLeAkIEyOcAwCG7XLCOczeUK7JDo0AyEMgqm68Ex9ZxGeDMHvq
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show keystore.seed
@qHfj*21kelFMztEFXLq
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password
pXoiWrJOQHWvy2OOlpJ_YQ
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.keystore.secure_password
MotF62oyTN659X-CuEHYyQ
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.truststore.secure_password
MotF62oyTN659X-CuEHYyQ
[root@localhost ~]# ls -l /etc/elasticsearch/certs/
total 24
-rw-rw----. 1 root elasticsearch  1915 Apr 25 16:17 http_ca.crt
-rw-rw----. 1 root elasticsearch 10013 Apr 25 16:17 http.p12
-rw-rw----. 1 root elasticsearch  5822 Apr 25 16:17 transport.p12
[root@localhost ~]# keytool -keystore /etc/elasticsearch/certs/http.p12 -list
Enter keystore password:
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 2 entries

http, Apr 25, 2023, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 5E:0E:5C:1C:8A:58:F6:0F:88:64:78:97:7F:FF:48:37:C2:25:70:33:36:DC:88:C0:F0:7B:28:5B:B6:47:A8:B4
http_ca, Apr 25, 2023, PrivateKeyEntry,
Certificate fingerprint (SHA-256): DE:CE:DD:17:1A:86:B7:90:F6:3C:AB:D5:D0:E2:4B:60:86:FD:75:8F:7B:E6:A3:3F:7A:10:4F:BE:4E:6D:78:F0
[root@localhost ~]# keytool -keystore /etc/elasticsearch/certs/transport.p12 -list
Enter keystore password:
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 2 entries

transport, Apr 25, 2023, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 9D:1B:31:2B:60:B1:31:DA:7C:6B:50:C8:27:8E:1E:D0:28:FF:85:2F:2B:FC:18:F0:01:83:93:87:D8:5E:84:BA
transport_ca, Apr 28, 2023, trustedCertEntry,
Certificate fingerprint (SHA-256): 02:4D:34:FE:92:C8:09:7A:8C:AC:D8:8E:73:20:78:D6:57:FA:9D:B9:43:25:23:B8:95:66:A9:AB:5C:75:3B:31
```

</details>

### 2.4. Generate certificates

References:

- <https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup.html>
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup-https.html>
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/update-node-certs-different.html>

The certificates can be created by `elasticsearch-certutil` or signed by a third party certificate authority, the examples below uses a self-signed lab certificate authority `central.pem` and `central.key`

Read more about generating self-signed certificate authority [here](https://github.com/joetanx/setup/blob/main/self-signed-ca.md)

- Elasticsearch provides `elasticsearch-certutil`, which creates RSA certicates
- Use `openssl` to create ECDSA certificates

<details><summary><h4>2.4.1. Generate RSA certificates with <code>elasticsearch-certutil</code></h4></summary>

##### Transport certificate

|Steps|Commands|
|---|---|
|Create CSR and output to `csr-bundle-transport.zip`|`echo csr-bundle-transport.zip \| /usr/share/elasticsearch/bin/elasticsearch-certutil csr`|
|Unzip CSR bundle to retrieve `instance.csr` and `instance.key`|`unzip /usr/share/elasticsearch/csr-bundle-transport.zip`|
|Create certificate|`openssl x509 -req -sha256 -days 1096 -CA central.pem -CAkey central.key -CAcreateserial -in instance.csr -out instance.pem`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out transport.p12 -inkey instance.key -in instance.pem -name transport -keysig -passout pass:elastic-transport`|
|Import the certificate authority into the certificate bundle with alias as `transport-ca`|`keytool -importcert -trustcacerts -noprompt -keystore transport.p12 -storepass elastic-transport -alias transport-ca -file central.pem`|
|Verify the resultant certificate bundle|`echo elastic-transport \| keytool -keystore transport.p12 -list`|

##### HTTP certificate

|Steps|Commands|
|---|---|
|Create CSR and output to `csr-bundle-http.zip`|`echo csr-bundle-http.zip \| /usr/share/elasticsearch/bin/elasticsearch-certutil csr --name foxtrot.vx --dns localhost,foxtrot.vx --ip 127.0.0.1,192.168.17.80`|
|Unzip CSR bundle to retrieve `foxtrot.vx.csr` and `foxtrot.vx.key`|`unzip /usr/share/elasticsearch/csr-bundle-http.zip`|
|Create config file for the SANs|`echo "subjectAltName=DNS:foxtrot.vx,DNS:localhost,IP:127.0.0.1,IP:192.168.17.80" > foxtrot.vx.cnf`|
|Create certificate|`openssl x509 -req -sha256 -days 1096 -CA central.pem -CAkey central.key -CAcreateserial -in foxtrot.vx.csr -extfile foxtrot.vx.cnf -out foxtrot.vx.pem`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out http.p12 -inkey foxtrot.vx.key -in foxtrot.vx.pem -name http -keysig -passout pass:elastic-http`|
|Import the certificate authority into the certificate bundle with alias as `http-ca`|`keytool -importcert -trustcacerts -noprompt -keystore http.p12 -storepass elastic-http -alias http-ca -file central.pem`|
|Verify the resultant certificate bundle|`echo elastic-http \| keytool -keystore http.p12 -list`|

</details>

#### 2.4.2. Generate ECDSA certificates with `openssl`

##### Transport certificate

|Steps|Commands|
|---|---|
|Create ECDSA key|`openssl ecparam -name secp384r1 -genkey -out transport.key`|
|Create CSR|`openssl req -new -key transport.key -subj "/CN=instance" -out transport.csr`|
|Create certificate|`openssl x509 -req -in transport.csr -CA central.pem -CAkey central.key -CAcreateserial -days 1096 -sha256 -out transport.pem`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out transport.p12 -inkey transport.key -in transport.pem -name transport -keysig -passout pass:elastic-transport`|
|Import the certificate authority into the certificate bundle with alias as `transport-ca`|`keytool -importcert -trustcacerts -noprompt -keystore transport.p12 -storepass elastic-transport -alias transport-ca -file central.pem`|
|Verify the resultant certificate bundle|`echo elastic-transport \| keytool -keystore transport.p12 -list`|

##### HTTP certificate

|Steps|Commands|
|---|---|
|Create ECDSA key|`openssl ecparam -name secp384r1 -genkey -out http.key`|
|Create CSR|`openssl req -new -key http.key -subj "/CN=foxtrot.vx" -out http.csr`|
|Create config file for the SANs|`echo "subjectAltName=DNS:foxtrot.vx,DNS:localhost,IP:127.0.0.1,IP:192.168.17.80" > http-openssl.cnf`|
|Create certificate|`openssl x509 -req -in http.csr -CA central.pem -CAkey central.key -CAcreateserial -days 1096 -sha256 -out http.pem -extfile http-openssl.cnf`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out http.p12 -inkey http.key -in http.pem -name http -keysig -passout pass:elastic-http`|
|Import the certificate authority into the certificate bundle with alias as `http-ca`|`keytool -importcert -trustcacerts -noprompt -keystore http.p12 -storepass elastic-http -alias http-ca -file central.pem`|
|Verify the resultant certificate bundle|`echo elastic-http \| keytool -keystore http.p12 -list`|

#### ☝️ About adding certificate authority to the `.p12` bundle using keytool

This step is required; otherwise, the below error will occur:

```
org.elasticsearch.ElasticsearchSecurityException: failed to load SSL configuration [xpack.security.transport.ssl] - the truststore [/etc/elasticsearch/certs/transport.p12] does not contain any trusted certificate entries
```

### 2.5. Configure Elasticsearch

#### 2.5.1. Replace the auto-generated certificates:

```sh
cp central.pem /etc/elasticsearch/certs/http_ca.crt
cp http.p12 /etc/elasticsearch/certs/http.p12
cp transport.p12 /etc/elasticsearch/certs/transport.p12
```

#### 2.5.2. Update keystore values:

```sh
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

#### 2.5.3. Edit Elasticsearch configuration file:

Backup original configuraton file: `cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.bak`

|Settings|Commands|
|---|---|
|Bind Elasticsearch to any address (optional)|`sed -i -e '/#network.host: 192.168.0.1/anetwork.bind_host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml`|
|Configure the stack as a single-node cluster|`sed -i -e '/#discovery.seed_hosts:/adiscovery.type: single-node' /etc/elasticsearch/elasticsearch.yml`|
|Change certificate verification mode to full|`sed -i 's/verification_mode: certificate/verification_mode: full/' /etc/elasticsearch/elasticsearch.yml`|
|Remove `cluster.initial_master_nodes` setting|`sed -i '/^cluster.initial_master_nodes:/d' /etc/elasticsearch/elasticsearch.yml`|

> Remove `cluster.initial_master_nodes` setting is required because setting `discovery.type: single-node` without removing `cluster.initial_master_nodes` will cause `java.lang.IllegalArgumentException: setting [cluster.initial_master_nodes] is not allowed when [discovery.type] is set to [single-node]` error

Enable + start Elasticsearch:

```sh
systemctl enable --now elasticsearch
```

### 2.6. Configure Kibana

#### 2.6.1. Copy certificates for Kibana

The same HTTP certificates generated for Elasticsearch are reused for Kibana since it's a single-node stack; generate a separate set of HTTP certificates as required

```sh
cp central.pem /etc/kibana/elasticsearch-ca.pem
mkdir /etc/kibana/certs
cp http.pem /etc/kibana/certs/http.crt
cp http.key /etc/kibana/certs/http.key
```

### 2.6.2. Reset password for `kibana_system` user and add the passwords to Kibana keystore

```sh
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
/usr/share/kibana/bin/kibana-keystore add elasticsearch.username
/usr/share/kibana/bin/kibana-keystore add elasticsearch.password
```

#### 2.6.3. Edit Kibana configuration file:

Backup original configuraton file: `cp /etc/kibana/kibana.yml /etc/kibana/kibana.yml.bak`

|Settings|Commands|
|---|---|
|Bind Kibana to any address|`sed -i -e '/#server.host: "localhost"/aserver.host: "0.0.0.0"' /etc/kibana/kibana.yml`|
|Set `publicBaseUrl`|`sed -i -e '/#server.publicBaseUrl:/aserver.publicBaseUrl: https:\/\/foxtrot.vx:5601' /etc/kibana/kibana.yml`|
|Enable Kibana HTTPS|`sed -i -e '/#server.ssl.enabled:/aserver.ssl.enabled: true' /etc/kibana/kibana.yml`|
|Set Kibana HTTPS certificate|`sed -i -e '/#server.ssl.certificate:/aserver.ssl.certificate: \/etc\/kibana\/certs\/http.crt' /etc/kibana/kibana.yml`|
|Set Kibana HTTPS key|`sed -i -e '/#server.ssl.key:/aserver.ssl.key: \/etc\/kibana\/certs\/http.key' /etc/kibana/kibana.yml`|
|Connect to Elasticsearch on localhost via HTTPS|`sed -i -e '/#elasticsearch.hosts:/aelasticsearch.hosts: https:\/\/localhost:9200' /etc/kibana/kibana.yml`|
|Set the Elasticsearch CA|`sed -i -e '/#elasticsearch.ssl.certificateAuthorities:/aelasticsearch.ssl.certificateAuthorities: \/etc\/kibana\/elasticsearch-ca.pem' /etc/kibana/kibana.yml`|
|Uncomment to enable full veritifcation of Elasticsearch certificate|`sed -i '/elasticsearch.ssl.verificationMode:/s/^#//' /etc/kibana/kibana.yml`|

Enable + start Kibana:

```sh
systemctl enable --now kibana
```

#### 2.6.4. Login to Kibana

Reset password for `elastic` and login

```sh
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### 2.7. Setup Elastic Agent

Ref: <https://www.elastic.co/guide/en/fleet/current/add-fleet-server-on-prem.html>

#### 2.7.1. Edit Elastic Agent output

Go to `Management` → `Fleet`, select `Edit` on the `default` entry under `Output`

1. Change the `Hosts` URL to `https` with the FQDN which the Elastic Agent will resolve to find Elasticsearch
2. Enter the `Elasticsearch CA trusted fingerprint`

To get certificate fingerprint: `openssl x509 -fingerprint -sha256 -in /path/to/elasticsearch-ca.crt | grep sha256 | sed 's/://g'`

![image](https://user-images.githubusercontent.com/90442032/235343709-edb47e12-b265-4a19-86a0-c74e070b2322.png)

#### 2.7.2. Setup Fleet Server

Go to `Management` → `Fleet`, select `Add Fleet Server` under `Fleet server hosts`

Enter Fleet Server details and select `Generate Fleet Server policy`

![image](https://user-images.githubusercontent.com/90442032/235343800-b5eae8e6-1c02-49cb-ad7e-1ef3332fd395.png)

The wizard provides Fleet Server installation commands after the policy is created:

![image](https://user-images.githubusercontent.com/90442032/235343943-a48f63f4-2611-49f7-acb7-bd35597670b6.png)

To use self-generated certificates, modify the `elastic-agent install` command as stated [here](https://www.elastic.co/guide/en/fleet/current/secure-connections.html#_encrypt_traffic_between_elastic_agents_fleet_server_and_elasticsearch):

```sh
sudo ./elastic-agent install \
   --url=https://192.0.2.1:8220 \
   --fleet-server-es=https://192.0.2.0:9200 \
   --fleet-server-service-token=AAEBAWVsYXm0aWMvZmxlZXQtc2XydmVyL3Rva2VuLTE2MjM4OTAztDU1OTQ6dllfVW1mYnFTVjJwTC2ZQ0EtVnVZQQ \
   --fleet-server-policy=fleet-server-policy \
   --fleet-server-es-ca-trusted-fingerprint=base64-sha256-fingerprint \
   --fleet-server-cert=/path/to/fleet-server.crt \
   --fleet-server-cert-key=/path/to/fleet-server.key
```

Example successful installation output:

```sh
Elastic Agent will be installed at /opt/Elastic/Agent and will run as a service. Do you want to continue? [Y/n]:
{"log.level":"info","@timestamp":"2023-04-30T16:20:04.753+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":753},"message":"Waiting for Elastic Agent to start","ecs.version":"1.6.0"}
{"log.level":"info","@timestamp":"2023-04-30T16:20:08.758+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":784},"message":"Fleet Server - Running on policy with Fleet Server integration: fleet-server-policy; missing config fleet.agent.id (expected during bootstrap process)","ecs.version":"1.6.0"}
{"log.level":"info","@timestamp":"2023-04-30T16:20:09.322+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":475},"message":"Starting enrollment to URL: https://foxtrot.vx:8220/","ecs.version":"1.6.0"}
{"log.level":"info","@timestamp":"2023-04-30T16:20:10.514+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":273},"message":"Successfully triggered restart on running Elastic Agent.","ecs.version":"1.6.0"}
Successfully enrolled the Elastic Agent.
Elastic Agent has been successfully installed.
```

![image](https://user-images.githubusercontent.com/90442032/235344182-83575a81-9584-4929-8cb4-8c54c339ad21.png)

Allow Fleet Server communication on firewall:

```sh
firewall-cmd --add-port 8220/tcp --permanent && firewall-cmd --reload
```

#### 2.7.3. Setup Elastic Agent

Select `Continue enrolling Elastic Agent` from the previous wizard and create an Agent policy:

![image](https://user-images.githubusercontent.com/90442032/235344236-d22fd826-6884-4656-9d0c-ac176db62df6.png)

The wizard provides Elastic Agent installation commands after the policy is created:

![image](https://user-images.githubusercontent.com/90442032/235344255-6604131d-02b9-4452-ba6b-87bb52c4e4c4.png)

To use self-generated certificates, modify the `elastic-agent install` command as stated [here](https://www.elastic.co/guide/en/fleet/current/secure-connections.html#_encrypt_traffic_between_elastic_agents_fleet_server_and_elasticsearch):

```sh
sudo elastic-agent install --url=https://192.0.2.1:8220 \
  --enrollment-token=<string> \
  --fleet-server-es-ca-trusted-fingerprint=base64-sha256-fingerprint
```

Example successful installation output:

```sh
Elastic Agent will be installed at /opt/Elastic/Agent and will run as a service. Do you want to continue? [Y/n]:
{"log.level":"info","@timestamp":"2023-04-30T16:23:07.968+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":475},"message":"Starting enrollment to URL: https://foxtrot.vx:8220/","ecs.version":"1.6.0"}
{"log.level":"info","@timestamp":"2023-04-30T16:23:08.822+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":273},"message":"Successfully triggered restart on running Elastic Agent.","ecs.version":"1.6.0"}
Successfully enrolled the Elastic Agent.
Elastic Agent has been successfully installed.
```

![image](https://user-images.githubusercontent.com/90442032/235344335-24154c78-b068-4417-a788-9f844d3bdc8f.png)

#### 2.7.4. Example Fleet Server + Elastic Agent output

![image](https://user-images.githubusercontent.com/90442032/235344386-a573adfc-ba08-44bf-ba2c-0875d9e6342e.png)

### 2.8. Integrate Suricata to Elastic

- How To Build A SIEM with Suricata and Elastic Stack on CentOS 8 Stream
  - <https://www.digitalocean.com/community/tutorials/how-to-build-a-siem-with-suricata-and-elastic-stack-on-centos-8-stream>
  - <https://www.howtoforge.com/how-to-install-and-configure-suricata-ids-along-with-elastic-stack-on-rocky-linux-8/>
- How To Create Rules, Timelines, and Cases from Suricata Events Using Kibana's SIEM Apps
  - <https://www.digitalocean.com/community/tutorials/how-to-create-rules-timelines-and-cases-from-suricata-events-using-kibana-s-siem-apps>

## 3. Host-based IDS: Wazuh

References:
- <https://documentation.wazuh.com/current/proof-of-concept-guide/index.html>

## 4. Performance monitoring: Grafana + Prometheus
