# 概述
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们部署的版本是此篇文章发布时ElastAlert（V0.2.1）、ElastAlert server（3.0.0-beta.0） 、elastalert-kibana-plugin（1.1.0）的最新版本。es和kibana的版本是7.2.0。ES和kibana部署不是本篇的重点，这里不做介绍。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;入正题之前得先说下ES，Elasticsearch 是一个分布式、可扩展、实时的搜索与数据分析引擎。 它能从项目一开始就赋予你的数据以搜索、分析和探索的能力。我们容器服务的日志现在大多数都入到了qbus，然后可以用es消费，通过kibana方便的查看日志。但是负责es的同事和我们都没有支持日志告警，目前是一个短板。X-Pack提供了报警组件Alert，但是这个功能是需要付费，ElastAlert能够完美替代Alert提供的所有功能。ElastAlert目前有6k+star，维护较好很受欢迎，使用python编写，V0.2.0以后使用python3，python2不再维护，以致在虚机物理机部署的坑很多，最终也是以容器的方式成功搭建。因为公司es之后要升级到7.2版本，ElastAlertV0.2.1（最新版本）才对es7支持的很好，elastalert-kibana-plugin最新版本才支持kibana7。
# 相关组件
* 告警服务ElastAlert，版本V0.2.1
* 展示插件elastalert-kibana-plugin，kibana安装这个组件后，用户可以通过kibana管理告警规则（增、删、改、查，及测试）。
* api服务ElastAlert  server，暴露restf api提供管理告警规则的能力，与elastalert-kibana-plugin配合使用。这个服务启动时同时启动ElastAlert。

elastalert-kibana-plugin对ES6.3开始提供支持，公司目前使用的是ES6.2，之后要升级到ES7，所以ElastAlert用V0.2.1，ElastAlert  server 3.0.0-beta.0,elastalert-kibana-plugin 1.1.0。
# 搭建
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ElastAlert V0.2.1需要python3，目前我们使用linux内核都会默认装python2，安装python3后，使用混合环境安装ElastAlert 很多依赖装不成功，亦或装成功后模块有找不到，所以最终打算使用docker multi stage build方式构建ElastAlert  server镜像，以容器的方式启动。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;官网上提供了ElastAlert  server镜像构建方式，但很久不维护，还是使用python2构建的老的版本。我们需要重新构建。
1. git clone [https://github.com/bitsensor/elastalert.git](https://github.com/bitsensor/elastalert.git) && cd elastalert  修改  Dockerfile。这一步很重要下载的是elastalert server，不是elastalert，elastalert只需在Dockerfile中指定版本，直接下载zip包。

```
FROM python:3.6-alpine as pyea
ENV ELASTALERT_VERSION=v0.2.1
# URL from which to download Elastalert.
ENV ELASTALERT_URL=https://github.com/Yelp/elastalert/archive/$ELASTALERT_VERSION.zip
# Elastalert home directory full path.
ENV ELASTALERT_HOME /opt/elastalert
WORKDIR /opt
RUN apk add --update --no-cache ca-certificates openssl-dev openssl libffi-dev gcc musl-dev wget && \
# Download and unpack Elastalert.
    wget -O elastalert.zip "${ELASTALERT_URL}" && \
    unzip elastalert.zip && \
    rm elastalert.zip && \
    mv e* "${ELASTALERT_HOME}"
WORKDIR "${ELASTALERT_HOME}"
# Install Elastalert.
# see: https://github.com/Yelp/elastalert/issues/1654
RUN python setup.py install && \
    pip install -r requirements.txt && \
    pip install elasticsearch==7.0.0
FROM node:alpine
#LABEL maintainer="BitSensor <dev@bitsensor.io>"
# Set timezone for this container
ENV TZ Etc/UTC
 
RUN apk add --update --no-cache curl tzdata python3 make libmagic
COPY --from=pyea /usr/local/lib/python3.6/site-packages /usr/lib/python3.6/site-packages
#COPY --from=pyea /usr/lib/python3.6/site-packages /usr/lib/python3.6/site-packages
COPY --from=pyea /opt/elastalert /opt/elastalert
COPY --from=pyea /usr/local/bin/elastalert* /usr/bin/
WORKDIR /opt/elastalert-server
COPY . /opt/elastalert-server
#RUN mkdir -p /usr/local/lib/python3.7 /usr/lib/python3.7 && \
#    cp /usr/local/lib/python3.6/site-packages /usr/local/lib/python3.7/site-packages\
#    cp /usr/local/lib/python3.6/site-packages /usr/lib/python3.7
RUN ln -s /usr/bin/python3 /usr/bin/python
RUN npm install --production --quiet
COPY config/elastalert.yaml /opt/elastalert/config.yaml
COPY config/elastalert-test.yaml /opt/elastalert/config-test.yaml
COPY config/config.json config/config.json
COPY rule_templates/ /opt/elastalert/rule_templates
COPY elastalert_modules/ /opt/elastalert/elastalert_modules
# Add default rules directory
# Set permission as unpriviledged user (1000:1000), compatible with Kubernetes
RUN mkdir -p /opt/elastalert/rules/ /opt/elastalert/server_data/tests/ \
    && chown -R node:node /opt
USER node
EXPOSE 3030
ENTRYPOINT ["npm", "start"]
```

2. 制作镜像：docker build --no-cache  -t   image-name  .

注：build的时候可能会报layer找不到的错误，再build一次就可以了（再次build时不要加--no-cache）（目前没有细跟原因）

3. 运行

```
docker run -d -p 3030:3030 -p 3333:3333 \
    -v `pwd`/config/elastalert.yaml:/opt/elastalert/config.yaml \
    -v `pwd`/config/elastalert-test.yaml:/opt/elastalert/config-test.yaml \
    -v `pwd`/config/config.json:/opt/elastalert-server/config/config.json \
    -v `pwd`/rules:/opt/elastalert/rules \
    -v `pwd`/rule_templates:/opt/elastalert/rule_templates \
    --net="host" \
    --name elastalert images-name
```

通过docker logs elastalert -f 查看日之日，如果报错，指定告警级别为dubug查看错误信息

4.  主要的配置文件：

* elastalert.yaml：elastalert配置文件，es相关配置

```

# This is the folder that contains the rule yaml files
# Any .yaml file will be loaded as a rule
rules_folder: rules
# How often ElastAlert will query Elasticsearch
# The unit can be anything from weeks to seconds
run_every:
  minutes: 1
# ElastAlert will buffer results from the most recent
# period of time, in case some log sources are not in real time
buffer_time:
  minutes: 15
# The Elasticsearch hostname for metadata writeback
# Note that every rule can have its own Elasticsearch host
es_host: 10.216.6.76
# The Elasticsearch port
es_port: 9200
es_username: tfmaq
es_password: 4152adc2a11cb182c014bc95
# The index on es_host which is used for metadata storage
# This can be a unmapped index, but it is recommended that you run
# elastalert-create-index to set a mapping
writeback_index: hihi_test
#writeback_alias: elastalert_alerts
# If an alert fails for some reason, ElastAlert will retry
# sending the alert until this time period has elapsed
alert_time_limit:
  days: 2
```

* config.json：elastalert server的配置文件

```

{
  "appName": "elastalert-server",
  "port": 3030,
  "wsport": 3333,
  "elastalertPath": "/opt/elastalert",
  "verbose": false,
  "es_debug": false,
  "debug": false,
  "rulesPath": {
    "relative": true,
    "path": "/rules"             # 通过kibana插件创建的配置文件都存在这
  },
  "templatesPath": {
    "relative": true,
    "path": "/rule_templates"
  },
  "es_host": "10.216.6.76",      # es ip
  "es_port": 9200,               # es 端口
  "writeback_index": "hihi_test" # 监控的esindex
}
```

* rules：所有报警规则的配置文件都存在着,test-rule.yaml

```

es_host: 10.216.6.76

es_port: 9200

es_username: tfmaq
es_password: 4152adc2a11cb182c014bc952xxx
name: Example frequency rule 2

type: frequency

index: hihi-test

num_events: 5

timeframe:
  hours: 4

filter:
- query:
   query_string:
     query: "type:access"  # 查询规则根据日志，自行配置
     query: "message:error"  #

alert:
- "email"

email:
- "zhangzhifei@360.cn" 
```

* 详细的配置规则大家看官方文档，照着我这么配置先把告警发出来

5. 安装kibana

按照网关按即可，版本要对上[https://github.com/bitsensor/elastalert-kibana-plugin](https://github.com/bitsensor/elastalert-kibana-plugin)

# kibana展示

kibana插件如下：
![image.png](https://upload-images.jianshu.io/upload_images/14774548-28d28f3c2e2a7876.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
custom_frequency是启服务之前创建的配置文件，test和test2是通过kibana dashboard创建。用户可以通过“创建”-“测试”-“保存”这一流程新建alert rules 和我们的hulk容器上的自定义告警流程差不多
![image.png](https://upload-images.jianshu.io/upload_images/14774548-ca24e86e65dfb74a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 参考

[https://github.com/bitsensor/elastalert](https://github.com/bitsensor/elastalert)

[https://github.com/bitsensor/elastalert-kibana-plugin](https://github.com/bitsensor/elastalert-kibana-plugin)

[https://github.com/Yelp/elastalert](https://github.com/Yelp/elastalert)

[https://elastalert.readthedocs.io/en/latest/](https://elastalert.readthedocs.io/en/latest/)


