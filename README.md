# remote-rsyslog-auth
Send logins to a syslog remotely
- Ubuntu 16.04 server
- rsyslog
- logstash
- elasticsearch
- kibana
- nginx with ssl

###### Prepare setup
```
wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

echo "deb https://packages.elastic.co/elasticsearch/2.x/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-2.x.list
echo "deb https://packages.elastic.co/logstash/2.3/debian stable main" | sudo tee -a /etc/apt/sources.list
echo "deb http://packages.elastic.co/kibana/4.5/debian stable main" | sudo tee -a /etc/apt/sources.list

add-apt-repository -y ppa:webupd8team/java
```

###### Install 
```
apt-get update
apt-get install elasticsearch logstash kibana nginx apache2-utils oracle-java8-installer -y
```

###### Config rsyslog
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
    host => "'$(hostname -f)'"
    port => "10514"
    codec => "json"
    type => "rsyslog"
  }
}

filter { }

output {
  if [type] == "rsyslog" {
    elasticsearch {
      hosts => [ "127.0.0.1:9200" ]
    }
  }
}' >> /etc/logstash/conf.d/logstash.conf 

sed -i.bak 's/# node.name: node-1/node.name: '$(hostname -f)'/g' /etc/elasticsearch/elasticsearch.yml 
sed -i 's/# cluster.name: my-application/cluster.name: elasticsearchCluster/g' /etc/elasticsearch/elasticsearch.yml 

sed -i.bak 's/# server.host: "0.0.0.0"/server.host: "127.0.0.1"/g' /opt/kibana/config/kibana.yml 
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
service logstash restart
service elasticsearch restart
service kibana restart
service nginx restart
```
