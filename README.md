## 1. Network IDS: Suricata

References:
- How To Install Suricata on CentOS 8 Stream
  - <https://www.digitalocean.com/community/tutorials/how-to-install-suricata-on-centos-8-stream>
  - <https://forum.suricata.io/t/guide-getting-started-on-centos-8-and-centos-7/538>
- Understanding Suricata Signatures
  - <https://www.digitalocean.com/community/tutorials/understanding-suricata-signatures>

```console
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
yum -y copr enable @oisf/suricata-latest
yum -y install suricata
cp /etc/suricata/suricata.yaml /etc/suricata/suricata.yaml.bak
sed -i 's/      community-id: false/\      community-id: true/' /etc/suricata/suricata.yaml
echo -e "detect-engine:\n  - rule-reload: true" >> /etc/suricata/suricata.yaml
sed -i 's/-i eth0/-i eth0 -i eth1/' /etc/sysconfig/suricata
systemtl daemon-reload
sudo -u suricata suricata-update
systemctl enable --now suricata
systemctl status suricata
sudo -u suricata suricata -T -c /etc/suricata/suricata.yaml -v
tail /var/log/suricata/suricata.log
curl http://testmynids.org/uid/index.html
curl -Lk http://testmyids.com
tail -f /var/log/suricata/fast.log
```

> Attempting to `curl https://testmyids.com` didn't log any events, perhaps because it's encrypted?

- More IDS test: <https://github.com/3CORESec/testmynids.org>
- Multiple interfaces: <http://pevma.blogspot.com/2015/05/suricata-multiple-interface.html>

## 2. SIEM: Elasticsearch + Kibana

References:
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/security-minimal-setup.html>
- <https://computingforgeeks.com/install-elastic-stack-elk-on-rhel-centos/>

```console
update-crypto-policies --set DEFAULT:SHA1
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
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
yum -y install --enablerepo=elasticsearch elasticsearch kibana
cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.bak
sed -i -e '/#network.host: 192.168.0.1/anetwork.bind_host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml
# sed -i -e '/#discovery.seed_hosts:/adiscovery.type: single-node' /etc/elasticsearch/elasticsearch.yml
# sed -i '/cluster.initial_master_nodes:/d' /etc/elasticsearch/elasticsearch.yml
firewall-cmd --permanent --add-service=elasticsearch
firewall-cmd --permanent --add-service=kibana
firewall-cmd --reload
systemctl enable --now elasticsearch
# /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
cp /etc/kibana/kibana.yml /etc/kibana/kibana.yml.bak
# KEYS=$(/usr/share/kibana/bin/kibana-encryption-keys generate -q --force)
# echo $KEYS >> /etc/kibana/kibana.yml
sed -i -e '/#server.host: "localhost"/aserver.host: "0.0.0.0"' /etc/kibana/kibana.yml
sed -i '/elasticsearch.hosts:/s/^#//g' /etc/kibana/kibana.yml
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
/usr/share/kibana/bin/kibana-verification-code
# /usr/share/kibana/bin/kibana-keystore add elasticsearch.username
# /usr/share/kibana/bin/kibana-keystore add elasticsearch.password
systemctl enable --now kibana
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### 2.1. Integrate Suricate to Elastic

- How To Build A SIEM with Suricata and Elastic Stack on CentOS 8 Stream
  - <https://www.digitalocean.com/community/tutorials/how-to-build-a-siem-with-suricata-and-elastic-stack-on-centos-8-stream>
  - <https://www.howtoforge.com/how-to-install-and-configure-suricata-ids-along-with-elastic-stack-on-rocky-linux-8/>
  - <https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html>
- How To Create Rules, Timelines, and Cases from Suricata Events Using Kibana's SIEM Apps
  - <https://www.digitalocean.com/community/tutorials/how-to-create-rules-timelines-and-cases-from-suricata-events-using-kibana-s-siem-apps>

## 3. Host-based IDS: Wazuh

References:
- <https://documentation.wazuh.com/current/proof-of-concept-guide/index.html>

## 4. Performance monitoring: Grafana + Prometheus
