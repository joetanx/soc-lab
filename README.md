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

### 2.2.0. Certificate configuration

<https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup-https.html>
<https://www.elastic.co/guide/en/elasticsearch/reference/current/certutil.html>
<https://www.elastic.co/guide/en/elasticsearch/reference/current/update-node-certs.html>

### 2.2.1. Configure Elasticsearch

Backup original configuraton file: `cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.bak`

The following commands configure:
- Bind Elasticsearch to `0.0.0.0`
- Configure the stack as a single-node cluster
- Remove `cluster.initial_master_nodes` setting

```sh
sed -i -e '/#network.host: 192.168.0.1/anetwork.bind_host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml
sed -i -e '/#discovery.seed_hosts:/adiscovery.type: single-node' /etc/elasticsearch/elasticsearch.yml
sed -i '/^cluster.initial_master_nodes:/d' /etc/elasticsearch/elasticsearch.yml
```

> `xpack.security.enabled: true` is set by default in Elasticsearch and Kibana from version 8.0.0 onwards
>
> Remove `cluster.initial_master_nodes` setting is required because setting `discovery.type: single-node` without removing `cluster.initial_master_nodes` will cause `java.lang.IllegalArgumentException: setting [cluster.initial_master_nodes] is not allowed when [discovery.type] is set to [single-node]` error

Enable + start Elasticsearch:

```sh
systemctl enable --now elasticsearch
```

### 2.2.2. Configure Kibana

Backup original configuraton file: `cp /etc/kibana/kibana.yml /etc/kibana/kibana.yml.bak`

The following commands configure:
- Bind Kibana to `0.0.0.0`
- Uncomment `#elasticsearch.hosts: ["http://localhost:9200"]` to connect Kibana to Elasticsearch on localhost

```sh
sed -i -e '/#server.host: "localhost"/aserver.host: "0.0.0.0"' /etc/kibana/kibana.yml
sed -i '/elasticsearch.hosts:/s/^#//g' /etc/kibana/kibana.yml
```

Enable + start Kibana:

```sh
systemctl enable --now kibana
```

### 2.2.3. Configure Kibana to connect to Elasticsearch

Generate enrollment token for Kibana: `/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana`

Browse to `http://kibana-host:5601` and enter the enrollment token:

![image](https://user-images.githubusercontent.com/90442032/234185217-440902aa-e4cb-4231-ad84-28d0249b1732.png)

Generate verification code: `/usr/share/kibana/bin/kibana-verification-code`

 Enter verification code into Kibana page:

![image](https://user-images.githubusercontent.com/90442032/234185269-2648ba42-8b6a-410d-8673-af15454349e5.png)

Reset password for `elastic` and login

```sh
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### 2.3. Integrate Suricata to Elastic

- How To Build A SIEM with Suricata and Elastic Stack on CentOS 8 Stream
  - <https://www.digitalocean.com/community/tutorials/how-to-build-a-siem-with-suricata-and-elastic-stack-on-centos-8-stream>
  - <https://www.howtoforge.com/how-to-install-and-configure-suricata-ids-along-with-elastic-stack-on-rocky-linux-8/>
- How To Create Rules, Timelines, and Cases from Suricata Events Using Kibana's SIEM Apps
  - <https://www.digitalocean.com/community/tutorials/how-to-create-rules-timelines-and-cases-from-suricata-events-using-kibana-s-siem-apps>

## 3. Host-based IDS: Wazuh

References:
- <https://documentation.wazuh.com/current/proof-of-concept-guide/index.html>

## 4. Performance monitoring: Grafana + Prometheus
