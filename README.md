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
