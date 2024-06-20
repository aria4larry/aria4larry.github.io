# Use Apache Superset and NiFi to create a data process and dashboard system.

## 1. Install Apache Superset
>Superset is a modern data exploration and data visualization platform. Superset can replace or augment proprietary business intelligence tools for many teams. Superset integrates well with a variety of data sources.

You can refer to this document for PyPI install: [Install Superset with PyPI](https://superset.apache.org/docs/installation/pypi). Basicly with following commands.

```sh
# virtualenv is shipped in Python 3.6+ as venv instead of pyvenv.
# See https://docs.python.org/3.6/library/venv.html
# Create and activate venv.
python3 -m venv venv
. venv/bin/activate
# Install and Initializing Superset
pip install apache-superset
superset db upgrade # this will initialize the database (default sqlite3, you can change database later)
# Create an admin user in your metadata database (use `admin` as username to be able to load the examples)
export FLASK_APP=superset # this env must set to start superset as it's a python module and a flask app at the same time.
superset fab create-admin

# Load some data to play with
superset load_examples # Optional

# Create default roles and permissions
superset init

# To start a development web server on port 8088, use -p to bind to another port
superset run -p 8088 --with-threads --reload --debugger # use develop server to start the server.
```
## 2. Config Superset
Normally we want superset connect to a prodction database like MySQL or PostgresQL.
We can use a ```superset_config.py``` to config it.

* superset_config.py
    ```python
    # superset_config.py
    from flask_limiter import Limiter
    from flask_limiter.util import get_remote_address
    import redis
    import logging
    
    # 配置 Redis 连接
    REDIS_HOST = 'localhost'
    REDIS_PORT = 6379
    REDIS_DB = 0
    REDIS_URL = f"redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}"
    
    # 初始化 Flask-Limiter 并使用 Redis 作为存储后端
    limiter = Limiter(
        key_func=get_remote_address,
        storage_uri=REDIS_URL
    )
    
    # 将 limiter 添加到 Superset 的扩展中
    def init_app(app):
        limiter.init_app(app)
    
    SECRET_KEY = "f8LkvdRW/p02l5KWn*****************lN5iA53Ge8KSwe9M2"
    SQLALCHEMY_DATABASE_URI = 'postgresql+psycopg2://username:password@db_host:5432/superset'
    
    # 确保 init_app 被调用
    #from superset.app import create_app
    #app = create_app()
    #init_app(app)
    
    # 添加日志输出
    logging.basicConfig(level=logging.INFO)
    logging.info("Loaded custom superset_config.py")
    ```

start command:
```sh
#!/bin/bash

# 设置环境变量
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
export SUPERSET_CONFIG_PATH=/home/user/Superset/superset_config.py
export FLASK_APP=superset

# 激活虚拟环境
source ./venv/bin/activate

# 启动 
superset run -p 8088 --with-threads --reload --debugger # use develop server to start the server.
```
## Run Superset with Gunicorn as a service
In prodction environment, we should use a production WSGI server like Gunicorn to serve the Superset Web APP.
## 3. Install Gunicorn
enter the venv and install Gunicorn
```shell
pip install gunicorn
```
## 4. Config gunicorn as service
To make gunicorn+ superset as a service, we can use systemd service to start/stop gunicorn(superset)

1. create and edit a service file
    ```sh
    sudo nano /etc/systemd/system/superset.service
    ```
    with following content
    ```ini
    [Unit]
    Description=Superset Gunicorn Service
    After=network.target
    
    [Service]
    User=user
    Group=user
    WorkingDirectory=/home/user/Superset
    Environment="LC_ALL=en_US.UTF-8"
    Environment="LANG=en_US.UTF-8"
    Environment="SUPERSET_CONFIG_PATH=/home/user/Superset/superset_config.py"
    Environment="FLASK_APP=superset"
    ExecStart=/home/user/Superset/venv/bin/gunicorn -w 4 -k gevent --worker-connections 1000 --timeout 120 -b 0.0.0.0:8088 --limit-request-line 0 --limit-request-field_size 0 "superset.app:create_app()"
    ExecReload=/bin/kill -s HUP $MAINPID
    ExecStop=/bin/kill -s TERM $MAINPID
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    ```

2. reload systemd config
    ```shell
    sudo systemctl daemon-reload
    ```

3. start and enable superset(service)
    ```shell
    sudo systemctl start superset
    sudo systemctl enable superset
    ```
4. you can use following command to check superset status
    ```shell
    sudo systemctl status superset
    ```

## 5. Install Apache Nifi
> Put simply, NiFi was built to automate the flow of data between systems. While the term 'dataflow' is used in a variety of contexts, we use it here to mean the automated and managed flow of information between systems. [Document](https://nifi.apache.org/documentation/v1/)
1. Download [Apache NiFi](https://nifi.apache.org/download/)
2. unzip the downloaded file
3. move to /opt/nifi `sudo mv nifi-1.26.0 /opt/nifi`
4. config env and start / view status
    ```shell
    export NIFI_HOME=/opt/nifi
    export PATH=$PATH:$NIFI_HOME/bin
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    nifi.sh start # start NiFi
    nifi.sh status # check status
    nifi.sh stop # stop NiFi
    ```
## 6. Run Nifi WebUI and config it remote accessable
nifi.properties file contains the configure.
```bash
sudo nano /opt/nifi/conf/nifi.properties
```
change the content
```ini
nifi.web.http.host= # set to blank or IP to remote access with IP.
nifi.web.http.port=8080 # http port.
#############################################
if in intranet, delete https settings, or else we need cetificate and create user to login NiFi WebUI
# nifi.web.https.host=
# nifi.web.https.port=8443
# nifi.web.https.network.interface.default=
# nifi.web.https.application.protocols=http/1.1
```


## 7. Superset Usage

## 8. NiFi Usage