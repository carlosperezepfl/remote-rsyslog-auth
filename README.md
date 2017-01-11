# remote-rsyslog-auth-sflow
Send logins to a syslog remotely and collect sFLOW
- Ubuntu 16.04 server
- rsyslog
- logstash
- elasticsearch
- kibana
- nginx with ssl

###### Prepare setup
```
- Install SSH keys
- NFS mount 
- Directory for the data and logs : 
      mkdir -p /data/data
      mkdir -p /data/logs
      chown -R elasticsearch:elasticsearch /data/
```

###### Install JAVA
```
apt-get install -y python-software-properties debconf-utils
add-apt-repository -y ppa:webupd8team/java
apt-get update
echo "oracle-java8-installer shared/accepted-oracle-license-v1-1 select true" | sudo debconf-set-selections
apt-get install -y oracle-java8-installer
```

###### Install ELK Stack
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elk.list

apt-get update && apt-get install -y elasticsearch logstash kibana

update-rc.d elasticsearch defaults 95 10
update-rc.d kibana defaults 95 10
```

###### Install sFLOW Codec
```
apt-get install -y ruby
git clone https://github.com/carlosperezepfl/logstash-codec-sflow
gem build logstash-codec-sflow/logstash-codec-sflow.gemspec 
/usr/share/logstash/bin/logstash-plugin install logstash-codec-sflow-2.0.0.gem 
```

###### Install nginx 
```
apt-get install -y nginx apache2-utils 
```


###### Config rsyslog for remote auth
```
sed -i.bak 's/#module(load="imudp")/module(load="imudp")/g' /etc/rsyslog.conf
sed -i 's/#input(type="imudp" port="514")/input(type="imudp" port="514")/g' /etc/rsyslog.conf
sed -i 's/#module(load="imtcp")/module(load="imtcp")/g' /etc/rsyslog.conf
sed -i 's/#input(type="imtcp" port="514")/input(type="imtcp" port="514")/g' /etc/rsyslog.conf

echo 'template(name="json-template"
  type="list") {
    constant(value="{")
      constant(value="\"@timestamp\":\"")     property(name="timereported" dateFormat="rfc3339")
      constant(value="\",\"@version\":\"1")
      constant(value="\",\"message\":\"")     property(name="msg" format="json")
      constant(value="\",\"sysloghost\":\"")  property(name="hostname")
      constant(value="\",\"severity\":\"")    property(name="syslogseverity-text")
      constant(value="\",\"facility\":\"")    property(name="syslogfacility-text")
      constant(value="\",\"programname\":\"") property(name="programname")
      constant(value="\",\"procid\":\"")      property(name="procid")
    constant(value="\"}\n")
} ' >> /etc/rsyslog.d/01-json-template.conf 


echo 'authpriv.*                      @'$(hostname -f)':10514;json-template' >> /etc/rsyslog.d/60-output.conf
echo 'auth.*                          @'$(hostname -f)':10514;json-template' >> /etc/rsyslog.d/60-output.conf
```

###### logstash
```
echo 'input {
  udp {
    port => 6343
    codec => sflow {}
    type => "sflow"
  }
  udp {
    host => "'$(hostname -f)'"
    port => "10514"
    codec => "json"
    type => "rsyslog"
  }
}

output {
        stdout { }
                if [type] == "sflow" {
                        elasticsearch {
                                index => "sflow-logstash-%{+YYYY.MM.dd}"
                                hosts => "localhost"
                        }
                }
                if [type] == "rsyslog" {
                        elasticsearch {
                                index => "rsyslog-logstash-%{+YYYY.MM.dd}"
                                hosts => "localhost"
                        }
                }
}' >> /etc/logstash/conf.d/logstash.conf 
```

###### elasticsearch
```
sed -i.bak 's/#node.name: node-1/node.name: '$(hostname -f)'/g' /etc/elasticsearch/elasticsearch.yml 
sed -i 's/#cluster.name: my-application/cluster.name: elasticsearchCluster/g' /etc/elasticsearch/elasticsearch.yml 
sed -i 's/#path.data: \/path\/to\/data/path.data: \/data\/data/g' /etc/elasticsearch/elasticsearch.yml 
sed -i 's/#path.logs: \/path\/to\/logs/path.logs: \/data\/logs/g' /etc/elasticsearch/elasticsearch.yml 
sed -i 's/#network.host: 192.168.0.1/network.host: 127.0.0.1/g' /etc/elasticsearch/elasticsearch.yml 
```
###### kibana
```
sed -i.bak 's/#server.host: "localhost"/server.host: "localhost"/g' /etc/kibana/kibana.yml
```

###### nginx + ssl
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt -subj "/C=CH/ST=VAUD/L=LAUSANNE/O=IC/OU=IC-IT/CN="$(hostname -f)


echo 'server {
listen      80;
server_name '$(hostname -f)';   
return 301 https://$server_name$request_uri;
}

server {
listen                	*:443 ;
ssl on;
ssl_certificate 		/etc/ssl/certs/nginx-selfsigned.crt;  
ssl_certificate_key 	/etc/ssl/private/nginx-selfsigned.key;  
server_name           	'$(hostname -f)'; 
access_log            	/var/log/nginx/kibana.access.log;
error_log  				/var/log/nginx/kibana.error.log;

location / {
auth_basic "Restricted";
auth_basic_user_file /etc/nginx/conf.d/kibana.htpasswd;
proxy_pass http://127.0.0.1:5601;
}
}' >> /etc/nginx/conf.d/kibana.conf 
```

###### nginx account
```
htpasswd -db -c /etc/nginx/conf.d/kibana.htpasswd admin xxxxxx
```

###### services
```
systemctl enable kibana.service 
systemctl enable elasticsearch.service 
systemctl enable logstash.service 
systemctl enable nginx.service 

service rsyslog restart
systemctl restart logstash.service
service elasticsearch restart
service kibana restart
service nginx restart
```

###### client setup
```
echo 'authpriv.*                      @SYSLOGSERVER:514' >> /etc/rsyslog.d/remote.conf
echo 'auth.*                          @SYSLOGSERVER:514' >> /etc/rsyslog.d/remote.conf
```
