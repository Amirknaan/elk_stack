# Elastic Stack Project

## Objective
setting up and configuring Elasticsearch, Kibana, Logstash, and Filebeat to ingest data from various sources, create dashboards, and configure index lifecycle management (ILM) and alerts.

## Prerequisites
Linux-based CentOS VM
Tools:
I worked with an AWS EC2 instance

## Instructions
1. Setting Up a Linux-based centOS VM<br>
I used this AMI ami-00543d76373f96fe7 to run on the EC2 instance.<br>
I Allocated appropriate resources (CPU, RAM, Disk space) and configured the needed Network settings and security groups by allowing inbound rules to the relevant ports:<br> 
80 and 443 for http and https<br>
22 for ssh into the machine<br>
9200 for elasticsearch<br>
5061 for kibana<br>
5044 for logstash.
2. install elasticsearch
import the GPG key
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

3. add the elastic repository
If you dont have vim, install it:<br>
sudo yum install vim<br>
sudo vim /etc/yum.repos.d/elasticsearch.repo<br>
and paste this:<br>
*[elasticsearch-8.x]*<br>
*name=Elasticsearch repository for 8.x packages*<br>
*baseurl=https://artifacts.elastic.co/packages/8.x/yum*<br>
*gpgcheck=1*<br>
*gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch*<br>
*enabled=1*<br>
*autorefresh=1*<br>
*type=rpm-md*

4. install elasticsearch
sudo yum install elasticsearch-8.1.3<br>
dont forget to save the password that is printed on the screen<br>
start and enable Elasticsearch<br>
sudo systemctl start elasticsearch<br>
sudo systemctl enable elasticsearch<br>

5. testing<br>
sudo systemctl status elasticsearch<br>
send a request:<br>
curl -kuelastic:password https://3.226.195.229:9200

6. install kibana<br>
sudo vim /etc/yum.repos.d/kibana.repo<br>
paste this:<br>
*[kibana-8.x]*<br>
*name=Kibana repository for 8.x packages*<br>
*baseurl=https://artifacts.elastic.co/packages/8.x/yum*<br>
*gpgcheck=1*<br>
*gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch*<br>
*enabled=1*<br>
*autorefresh=1*<br>
*type=rpm-md*

sudo yum install kibana<br>
enter the file at /etc/kibana/kibana.yml and change the server.host value to "0.0.0.0"<br>
sudo systemctl start kibana<br>
sudo systemctl enable kibana<br>

7. configure kibana<br>
enter on the browser to the public ip, in my case it's http://3.226.195.229:5601<br>
you will be asked to provide an enrollment token.<br>
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token<br>
paste the token in the browser, and than generate the verification code like this:<br>
sudo /usr/share/kibana/bin/kibana-verification-code.<br>
to authenticate insert the default username "elastic" and the password you kept from the elastic search installation.

8. Install logstash<br>
sudo vim /etc/yum.repos.d/logstash.repo<br>
paste:<br>
*[logstash-8.x]*<br>
*name=Logstash repository for 8.x packages*<br>
*baseurl=https://artifacts.elastic.co/packages/8.x/yum*<br>
*gpgcheck=1*<br>
*gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch*<br>
*enabled=1*<br>
*autorefresh=1*<br>
*type=rpm-md*

sudo yum install logstash-8.1.3<br>
sudo systemctl start logstash<br>
sudo systemctl enable logstash<br>
sudo systemctl status logstash<br>

9. Install filebeat<br>
sudo vim /etc/yum.repos.d/filebeat.repo<br>
paste:<br>
*[filebeat-8.x]*<br>
*name=Filebeat repository for 8.x packages*<br>
*baseurl=https://artifacts.elastic.co/packages/8.x/yum*<br>
*gpgcheck=1*<br>
*gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch*<br>
*enabled=1*<br>
*autorefresh=1*<br>
*type=rpm-md*

sudo yum install filebeat-8.1.3<br>
sudo systemctl start filebeat<br>
sudo systemctl enable filebeat<br>
sudo systemctl status filebeat<br>

10. Ingest CEF data with Logstash<br>
logstash.conf file attached in the github repo<br>
sudo systemctl restart logstash<br>
create an Index template for CEF data - the template file name is ecs_template.json and it's available in the github repo<br>
apply it with a put request curl -kuelastic:password PUT "http://public-ip:9200_index_template/json_expm_template" -H 'Content-Type: application/json' -d @json_expm_template.json

11. create index pattern<br>
go to kibana -> Data views -> add field and choose the filter you wish, and click save.

12. Create Dashboard<br>
Dashboards -> create Dashboard -> create virtualization -> choose the data view and available fields + the virtualization ( for example Bar vertical stacked ) and save.<br>
![logo](/home/amir/Pictures/Screenshots/dashboard.png)

13. Define an ILM<br>
Before defining an ILM we need to define an alias, which is required for rollover.<br>
curl -kuelastic:password PUT "http://publicip:9200/my-index" -H 'Content-Type: application/json' -d'<br>
{
  "aliases": {
    "my-index-alias": {
      "is_write_index": true
    }
  }
}'


than at the browser go to managment -> index life cycle policies and choose the policy you wish, in our case keeping the data 30 days or rollup at 50g. attach the alias you created.

14. explain the differences and the decision making in putting an Index at cold/Frozen/Closed.<br>
let's explain what every state does.<br>
closed:<br>
Not available for either read or write requests. The Index isnt loaded to the memory and doesnt consume RAM or CPU.
It can be used when there is no frequent need for data, like archiving.<br>
frozen:<br>
Index that defined as frozen is kept in the system but loads to memory only when a query is made.<br>
Can be useful if the data doesn't require high performance or infrequently needed, and also in order to save resources.<br>
cold:<br>
Index defined as "cold" will be accessible for read-only, and kept on a slower perforamance hardware.<br>
Useful if the performance isn't critical and there is no need to update the data-just store it. cheaper hardware is an advantage here.

the main three parameters to decide in which state we should define an index will be:<br>
1. how frequent the data is accessed
2. how important the performance of retrieving data
3. how the index impacts the resource consumption<br>
for best performance we will define the index as "hot"

## Filebeat & Logstash
1. Define in /etc/filebeat/filebeat.yml the host of kibana, logstash and a path to the file
2. Edit logstash.conf 
3. create an index template ( file json_expm_template.json )
apply it with a put request curl -kuelastic:password PUT "http://public-ip:9200_index_template/json_expm_template" -H 'Content-Type: application/json' -d @json_expm_template.json
4. create index pattern and ILM same as you did in stages 7 and 9 in the previous drill
5. Define an alert that way:
In kibana enter at the scroll bar to managment -> Rules and connectors -> create rule 
and there you should define the alerts you wish and the actions ( the only two that allowed without gold license is Index and server log - I did both)`
