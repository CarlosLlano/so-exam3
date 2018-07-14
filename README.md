# PARCIAL 3 DEL CURSO SISTEMAS OPERATIVOS #




***1. Consignar las instrucciones comentadas para la instalación y configuración del stack de sensu (cliente, servidor, api, uchiwa dashboard y 
rabbitmq como bus de mensajería)***

A continuacion se explica como instalar y configurar un stack de sensu, cuya configuracion basica presenta los elementos resaltados 
en el cuadro rojo de la siguiente imagen:

![sensu](https://cloud.githubusercontent.com/assets/17281733/26524709/d072b494-4301-11e7-87d9-832437b4ef08.png)

***Desde el lado del servidor***, un sistema de monitoreo de sensu cuenta con los siguientes elementos:
  
1.*sensu-server*: se encarga de publicar checks, para realizar monitoreo, a los cuales los clientes se subscriben. 
Los clientes ejecutan estos checks y envian los resultados. Sensu-server los almacena en una base de datos llamada redis.

2.*redis*: es una base de datos que se utiliza en sensu para guardar la informacion enviada por los clientes en formato JSON.

3.*sensu-api*: API REST utilizada para acceder a los datos de monitoreo de sensu

4.*sensu-dashboard*: Se utiliza para mostrar de forma grafica los resultados del monitoreo. En este caso se utilizara el dashboard uchiwa.

5.*rabbitmq*: es un bus de mensajeria, que recibe todos los resultados de los clientes y los suministra uno por uno al componente
sensu-server. Su principal funcion es evitar saturar el componente sensu-server con gran cantidad de resultados de monitoreo 
provenientes de los clientes.

Para la instalacion de estos componentes en el servidor, se deben ejecutar los siguientes comandos.

```bash
echo '[sensu]
name=sensu
baseurl=https://sensu.global.ssl.fastly.net/yum/$releasever/$basearch/
gpgcheck=0
enabled=1' | sudo tee /etc/yum.repos.d/sensu.repo

yum install sensu -y
su -c 'rpm -Uvh http://download.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-9.noarch.rpm'
yum install erlang -y
yum install redis -y
yum install socat -y
su -c 'rpm -Uvh http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.9/rabbitmq-server-3.6.9-1.el7.noarch.rpm'
service rabbitmq-server start
rabbitmqctl add_vhost /sensu
rabbitmqctl add_user sensu password
rabbitmqctl set_permissions -p /sensu sensu ".*" ".*" ".*"
rabbitmq-plugins enable rabbitmq_management
chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
yum install uchiwa -y
firewall-cmd --zone=public --add-port=5672/tcp --permanent   
firewall-cmd --zone=public --add-port=15672/tcp --permanent
firewall-cmd --zone=public --add-port=3000/tcp --permanent
firewall-cmd --reload
```

Cada componente mencionado cuenta con un archivo de configuracion en formato json, cuya ubicacion es /etc/sensu/conf.d. De esta forma:

***para sensu-server ---> client.json***
```bash
{
  "client": {
    "name": "server",
    "address": "127.0.0.1",
    "subscriptions": [ "ALL" ]
  }
}
```

***para redis ---> redis.json***
```bash
{
  "redis": {
    "host": "127.0.0.1",
    "port": 6379
  }
}
```

***para sensu-api ---> api.json***
```bash
{
  "api": {
    "host": "127.0.0.1",
    "bind": "0.0.0.0",
    "port": 4567
  }
}
```

***para rabbitmq ---> rabbitmq.json***
```bash
{
  "rabbitmq": {
    "host": "127.0.0.1",
    "port": 5672,
    "vhost": "/sensu",
    "user": "sensu",
    "password": "password"
  }
}
```

***para uchiwa ---> uchiwa.json***
```bash
{
    "sensu": [
        {
            "name": "sensu",
            "host": "localhost",
            "ssl": false,
            "port": 4567,
            "path": "",
            "timeout": 5000
        }
    ],
    "uchiwa": {
        "port": 3000,
        "stats": 10,
        "refresh": 10
    }
}
```

Por otro lado, como ya se ha mencionado antes, el servidor de sensu se encarga de publicar checks a los cuales los clientes se 
subscriben. Para publicar estos checks hay que definir un archivo en formato json. En este caso, se definira un check para comprobar 
que un servicio web (documentado en el siguiente enlace: https://github.com/CarlosLlano/so-exam2/blob/master/A00170892.md) este funcionando.

Asi, el archivo check_apache.json define:

```bash
{
  "checks": {
    "servicioWeb_check": {
      "command": "/opt/sensu/embedded/bin/ruby /etc/sensu/plugins/check_web.rb",
      "interval": 15,
      "subscribers": ["webservers"]
    }
  }
}
```

El check establece que cada cliente, si se subscribe a "webservers", debe ejecutar el comando definido en "command".
El comando consiste en ejecutar un script de ruby, por tanto, cada cliente suscrito debe tener dicho script
en la carpeta /etc/sensu/plugins. Dicho script es el siguiente:

```ruby
procs = `ps aux`
running = false
procs.each_line do |proc|
  running = true if proc.include?('python web.py')
end
if running
  puts 'OK - Api rest funcionado'
  exit 0
else
  puts 'WARNING - Api rest no esta funcionando'
  exit 1
end
```

***Desde el lado del cliente***, basta con instalar el paquete de monitoreo de sensu con los siguientes comandos:

```bash
echo '[sensu]
name=sensu
baseurl=https://sensu.global.ssl.fastly.net/yum/$releasever/$basearch/
gpgcheck=0
enabled=1' | sudo tee /etc/yum.repos.d/sensu.repo

yum install sensu -y
```

Los archivos de configuracion son archivos en formato json almacenados en la carpeta /etc/sensu/conf.d

***para identificar al cliente ante el servidor ---> client.json***
```bash
{
  "client": {
    "name": "cliente-parcial3",
    "address": "192.168.0.19",
    "subscriptions": ["webservers"]
  }
}
```

***para identificar al servidor ---> rabbitmq.json***
```bash
{
  "rabbitmq": {
    "host": "192.168.0.21",
    "port": 5672,
    "vhost": "/sensu",
    "user": "sensu",
    "password": "password",
    "heartbeat": 10,
    "prefetch": 50
  }
}
```


Para poner en funcionamiento el servidor, ejecutar en el siguiente orden:
```bash
service redis start
service rabbitmq-server start
service sensu-server start & service sensu-api start
service uchiwa start  
```

Para poner en funcionamiento el cliente, ejecutar
```bash
service sensu-client start
```

***2. Evidencie la correcta instalación del stack de sensu por medio de la creación de un api rest de prueba y su check de sensu respectivo. 
Evidencie la generación de una alerta ante la caída del servicio***


El api rest de prueba que se utilizo fue el implementado para el parcial 2 (https://github.com/CarlosLlano/so-exam2/blob/master/A00170892.md)
Como se menciono antes, se definio un check que valida si este servicio esta funcionando o no.

![prueba](https://cloud.githubusercontent.com/assets/17281733/26525141/54053d98-4313-11e7-86ad-81a9c2a07d18.gif)


***3. Realice la instalación y configuración de un plugin de sensu para el monitoreo de infraestructura (disco, memoria ram, procesador)***

se instalo un plugin de sensu para el monitoreo de cpu:

![1 sensu-install](https://cloud.githubusercontent.com/assets/17281733/26525419/cce7f5a4-431b-11e7-9f1e-29e7770d7492.jpeg)

![2 gemas](https://cloud.githubusercontent.com/assets/17281733/26525422/d7958e44-431b-11e7-8e37-08e36563f963.jpeg)

![3 checks](https://cloud.githubusercontent.com/assets/17281733/26525423/e4024be0-431b-11e7-8d56-cb8633b0f620.jpeg)

![4 resultados](https://cloud.githubusercontent.com/assets/17281733/26525424/ece5b2ce-431b-11e7-8ba4-19613d26e1e5.jpeg)

Se utilizará para la prueba el archivo check-load.rb. Para mayor facilidad, se hara una copia en /etc/sensu/plugins.
Tambien se modifico el archivo de configuracion de checks del servidor:

```bash
{
  "checks": {
    "servicioWeb_check": {
      "command": "/opt/sensu/embedded/bin/ruby /etc/sensu/plugins/check-load.rb",
      "interval": 15,
      "handlers": ["slack"],
      "subscribers": ["webservers"]
    }
  }
}
```
Evidencia del funcionamiento:

![evidencia](https://cloud.githubusercontent.com/assets/17281733/26525532/87dc6a68-431f-11e7-9e47-dc01adb7f6c9.jpeg)


***4. Crear un handler en slack para el envío de una notificación en caso de ocurrir una caída del servicio de prueba creado
en el ítem anterior.***


El servicio de prueba sera el servicio web utilizado anteriormente. Para el handler de slack se crea el archivo 
handler_slack.json en el servidor.

```bash
{
    "handlers": {
        "slack": {
            "type": "pipe",
            "command": "/opt/sensu/embedded/bin/ruby /opt/sensu/embedded/lib/ruby/gems/2.4.0/gems/sensu-plugins-slack-1.0.0/bin/handler-slack.rb -j slack-integration",
            "severites": ["warning", "unknown"]
        }
    },
    "slack-integration": {
        "webhook_url": "https://hooks.slack.com/services/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",
        "template" : ""
    }
}
```

En el archivo de checks del servidor se añade este handler de slack.

```bash
{
  "checks": {
    "servicioWeb_check": {
      "command": "/opt/sensu/embedded/bin/ruby /etc/sensu/plugins/check_web.rb",
      "interval": 15,
      "handlers": ["slack"],
      "subscribers": ["webservers"]
    }
  }
}
```

Anteriormente se creo un team llamado "TestSO", al cual se le creo un canal llamado "service-alerts". De dicho canal
se obtuvo el webhook_url presente en handler_slack.json.

Evidencia del funcionamiento:

![evidencia slack](https://cloud.githubusercontent.com/assets/17281733/26525607/364eeea2-4322-11e7-8081-ca0a03bb690e.jpeg)


***5. Consignar las instrucciones comentadas para la instalación y configuración del stack de ELK (elasticsearch, logstash, kibana, 
filebeat en el client)***

En este caso se utilizara un solo nodo para la instalacion de todo el stack ELK.
El primer paso es instalar la llave publica

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

Posteriormente, se procede a instalar cada uno de los componentes del stack. Para cada uno se crea un archivo que servira
de repositorio y luego se procede a la instalacion.

***elasticseach***

repositorio
```bash
vi /etc/yum.repos.d/elasticsearch.repo
```

contenido del repositorio
```bash
[elasticsearch-5.x] 
name=Elasticsearch repository for 5.x packages 
baseurl=https://artifacts.elastic.co/packages/5.x/yum gpgcheck=1 
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch 
enabled=1 autorefresh=1 type=rpm-md
```

instalacion
```bash
sudo yum install elasticsearch
```

***logstash***

repositorio
```bash
vi /etc/yum.repos.d/logstash.repo
```

contenido del repositorio
```bash
[logstash-5.x] 
name=Elastic repository for 5.x packages 
baseurl=https://artifacts.elastic.co/packages/5.x/yum gpgcheck=1 
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch 
enabled=1 autorefresh=1 type=rpm-md
```

instalacion
```bash
sudo yum install logstash
```

***kibana***

repositorio
```bash
vi /etc/yum.repos.d/kibana.repo
```

contenido del repositorio
```bash
[kibana-5.x] 
name=Kibana repository for 5.x packages 
baseurl=https://artifacts.elastic.co/packages/5.x/yum gpgcheck=1 
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch 
enabled=1 autorefresh=1 type=rpm-md
```

instalacion
```bash
sudo yum install kibana
```

***filebeat**

repositorio
```bash
vi /etc/yum.repos.d/elastic.repo
```

contenido del repositorio
```bash
[elastic-5.x]
name=Elastic repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1 autorefresh=1 type=rpm-md
```

instalacion
```bash
sudo yum install filebeat
```

Una vez terminada la instalacion, se debe proceder a la configuracion de cada uno de los elementos del stack.
Para ello, es importante tener clara las funcionalidades de cada elemento:

*elasticsearch* = base de datos no relacional donde se guardan datos en formatos json

*Kibana* = herramienta de visualizacion de los datos de elasticsearch

*filebeat* = es un intermediario, envia logs desde el cliente hacia logstash.

*logstash* = es un collector, sirve para preprocesar la información (logs) enviada por los clientes antes de enviarla a elasticsearch 


Dicho esto, las configuraciones son las siguientes:

***para elasticsearch**

En el archivo /etc/elasticsearch/elasticsearch.yml.

```bash
network.host: 192.168.0.19
http.port: 9200
```

***para kibana***

En el archivo /etc/kibana/kibana.yml.

```bash
server.port: 5601
server.host: "192.168.0.19"
elasticsearch.url: "http://192.168.0.19:9200"
```

***para filebeat***

Dado que filebeat debe enviar logs, se debe especificar cual es el archivo que contiene esos logs. En este
caso se utilizara el archivo de logs de httpd

En el archivo /etc/filebeat/filebeat.yml

```bash
input_type: log
paths:
    - /var/log/httpd/access_log
```

Tambien se debe especificar el destino de esos logs.

```bash
output.logstash:
  hosts: ["192.168.0.19:5044"]
```

***para logstash***

Dado que logstash debe preprocesar una informacion antes de enviarla a elasticsearch, se debe especificar en un archivo
cual es la entrada de la informacion (input), la manera de preprocesarla (filter), y una salida (output).

![logstash basics](https://cloud.githubusercontent.com/assets/17281733/26525790/113d1304-4328-11e7-8df0-a469fbe16eb5.jpeg)


Para ello se crea el archivo /etc/logstash/conf.d/apache-logstash.conf con el contenido siguiente:

```bash
input {
    beats {
        port => "5044"
    }
}
filter {
    grok {
        match => { "message" => "%{COMBINEDAPACHELOG}"}
    }
}
output {
    elasticsearch
    {
        hosts => ["192.168.0.19:9200"]
    }
}
```

El objetivo de este filtro en particular es preprocesar los logs de httpd, los cuales tienen una estructura en cierto grado desordenada...

![log](https://cloud.githubusercontent.com/assets/17281733/26525816/c918f6d2-4328-11e7-9539-49019b729c29.jpeg)


...y darles un formato en el cual sean identificables los siguientes campos.

![imagen](https://cloud.githubusercontent.com/assets/17281733/26525820/fa74dcb4-4328-11e7-9995-c028691428b2.jpeg)


Finalmente para poner todo en funcionamiento se deben ejecutar los siguientes comandos:

```bash
service elasticsearch start
service kibana start
service logstash start
filebeat.sh -e -c filebeat.yml -d "publish"
```


***6. Evidencie la correcta instalación del stack ELK***


![kibana1](https://cloud.githubusercontent.com/assets/17281733/26525887/e474e678-432a-11e7-81dc-e3085daf5d9e.jpeg)


![kibana2](https://cloud.githubusercontent.com/assets/17281733/26525880/c71f04f0-432a-11e7-99a8-5def5ca25cb5.jpeg)


![kibana3](https://cloud.githubusercontent.com/assets/17281733/26525881/c85ed340-432a-11e7-9bde-46d0a43c45c9.jpeg)

