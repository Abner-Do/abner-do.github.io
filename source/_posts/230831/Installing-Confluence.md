---
title: Confluence
date: 2022-08-21 17:40:47
tags:
---

## Installing confluence

To run a Confluence application, you can use docker-compose.

Here are the steps to follow:

### 1. Generate a License Key
   
The first step is to run the `install_confluence.sh` script provided in the document. This script will build the necessary image, run a container with the image, and generate a license key. The license key is then stored in a `.env` file, which will be used by the `docker-compose.yml` file to set the `ATL_LICENSE_KEY` environment variable for the `confluence` service.

```
#!/bin/bash

docker build -t atlassian/confluence-server:8.2.0.local ./confluence

docker run \
    -d \
    --name="confluence-key" \
    -v confluence-key:/var/atlassian/application-data/confluence \
    atlassian/confluence-server:8.2.0.local

#init confluence.cfg.xml file.
docker exec confluence-key curl -L localhost:8090 2>&1 > /dev/null

#get server id.
SERVER_ID=`docker exec confluence-key cat /var/atlassian/application-data/confluence/confluence.cfg.xml | grep setup.server.id | awk -F'[<>]' '{print $3}'`

#genarate license key.
ATL_LICENSE_KEY=$(docker exec confluence-key \
 java -jar /var/atlassian/application-data/confluence/atlassian-agent.jar \
 -d \
 -m Linux@confluence.com \
 -n confluence-`date +%y%m%d` \
 -p conf \
 -o http://localhost \
 -s $SERVER_ID| sed -n '10,18p'|tr -d '\n')
echo -e $ATL_LICENSE_KEY
echo -e "ATL_LICENSE_KEY=$ATL_LICENSE_KEY" > .env
#docker-compose up -d
#docker stop confluence-key && docker rm confluence-key
```

### 2. Starting the Confluence Application
After running the install_confluence.sh script and creating the docker-compose.yml file, you can start the Confluence application by running the command docker-compose up -d in the same directory as the docker-compose.yml file. This will start the confluence and postgres services, allowing you to access the Confluence application at http://localhost:8090. If you need to stop the application, you can run the command docker-compose down in the same directory as the docker-compose.yml file.

```
version: '3.4'

services:
    confluence:
        container_name: confluence
        image: atlassian/confluence-server:8.2.0.local
        environment:
            - ATL_LICENSE_KEY
            - ATL_JDBC_URL=jdbc:postgresql://db:5432/confluence
            - ATL_JDBC_USER=confluenceuser
            - ATL_JDBC_PASSWORD=xxxx
            - ATL_DB_TYPE=postgresql
            - ATL_PROXY_NAME=linuxsa-confluence.brbiotech.tech  
            - ATL_TOMCAT_SCHEME=https
            - ATL_PROXY_PORT=443
        #   - JVM_MINIMUM_MEMORY=1g
        #   - JVM_MAXIMUM_MEMORY=12g
        #   - JVM_CODE_CACHE_ARGS='-XX:InitialCodeCacheSize=1g -XX:ReservedCodeCacheSize=8g'
        build:
            context: ./confluence
            #args:
            #    JVM_SUPPORT_RECOMMENDED_ARGS="-javaagent:/var/atlassian/application-data/confluence/atlassian-agent.jar"
        links:
            - postgres:db
        depends_on:
            - postgres
        ports:
            - "8090:8090"
            - "8091:8091"
        volumes:
            - confluence:/var/atlassian/application-data/confluence
        restart: always
        networks:
            - network-bridge


    postgres:
        container_name: postgres
        build:
            context: ./database/postgres
        image: postgres:15.2.local
        restart: always
        #environment:
        #    - TZ=Asia/Shanghai
        #    - PGDATA: /var/lib/postgresql/data/pgdata
        volumes:
            - postgres:/var/lib/postgresql/data/pgdata
        ports:
            - "5432:5432"
        networks:
          - network-bridge

volumes:
    confluence:
        external: false
        name: confluence
    postgres:
        external: false
        name: postgres

networks:
    network-bridge:
        driver: bridge
```

### 3. Configure database and set admin user
To set up an admin user, log in to Confluence application using a web browser at the address http://ip:8090. Then, follow the instructions in the setup wizard to create an admin user.

![admin user](shortcut.png)

### 4. Reference

For more information on installing and configuring Confluence with Docker, please refer to the following resources:

- Third-party Confluence installation: https://github.com/haxqer/confluence
- Atlassian Confluence Docker image: https://hub.docker.com/r/atlassian/confluence-server
- PostgreSQL Docker image: https://hub.docker.com/_/postgres
- Database setup for PostgreSQL: https://confluence.atlassian.com/doc/database-setup-for-postgresql-173244522.html

### 5. Others

postgres host auth method in docker container:

/var/lib/postgresql/data/pg_hba.conf

Useful postgresql commands as beblow:

```bash
ALTER USER postgres WITH PASSWORD 'xxx';
\l;
\-c [database_name];
\dt;
```

### 6. Maintainer

Are you experiencing any issues with Confluence? Our team is here to help! As the maintainer of this Confluence installation, I encourage you to reach out to us if you have any problems or concerns. You can contact us at yu_jia[@aliyun.com](mailto:jia.yu@brbiotech.com), and we will do our best to assist you in resolving any issues you may be facing. Don't hesitate to reach out to us - our goal is to ensure that your experience with Confluence is as smooth and trouble-free as possible.