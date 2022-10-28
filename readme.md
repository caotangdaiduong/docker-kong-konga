Setup (Kong + Konga) as API Gateway
===================================

[#kong](/t/kong) [#docker](/t/docker) [#tutorial](/t/tutorial) [#konga](/t/konga)

Welcome! You are about to start on a journey about and how to setup kong as an API Gateway for your infrastructure. In this second chapter, We are going to learn how to setup Kong and Konga. By the end of this series you are going to have your Kong and Konga running.

[](#table-of-content)Table of Content
=====================================

*   [Api Gateway Series](#api-gateway-series)
*   [Prerequisites](#prerequisites)
*   [Kong](#konga)
*   [Konga](#konga)
*   [Repository](#repository)
*   [Support](#support)

In this articles, i won't talk to much about why choose kong compare to another product. Tl;dr is because kong is open-source, has good community thanks to [Kong Hub](https://docs.konghq.com/hub/) and it can be easily run using Docker because they have prebuilt [image](https://hub.docker.com/_/kong).

[](#prerequisites)Prerequisites
-------------------------------

Before jumping to how, you need to make sure you have this installed:

*   [Docker](https://www.docker.com/get-started)
*   [Docker Compose](https://docs.docker.com/compose/)

[](#kong)Kong
-------------

To run kong via docker-compose we need to prepare docker-compose (Kong + DB) because we want to run kong using database. There's 2 database that currently supported by kong (PostgreSQL + Cassandra). In this articles i'm going to configure it using PostgreSQL. First let's configure Postgres services.

Dockerfile  

    FROM postgres:9.6
    
    COPY docker-entrypoint.sh /usr/local/bin/
    
    ENTRYPOINT ["docker-entrypoint.sh"]
    CMD ["postgres"]
    

Enter fullscreen mode Exit fullscreen mode

If you're wondering why would we create our own postgres dockerfile instead directly using prebuilt images is because we need to load custom docker-entrypoint.sh which support create multiple initial user + database that later going to be used by (Kong + Konga), compare to prebuilt images where you can only create one user + one database. With this now our postgres can set environment like this:  

    environment:
     POSTGRES_USERS: username:password | username2:password 
     POSTGRES_DATABASES: dbname:username | dbname2:username2
    

Enter fullscreen mode Exit fullscreen mode

Now let's continue wrote our docker-compose.yml, let's create services called db and kong-migrations. In db we build from context instead of images, to make all of our container portable and prevent _hardcoded_ config we make environment to refer our own defined variable called KONG\_DB\_USERNAME, KONG\_DB\_PASSWORD, KONGA\_DB\_USERNAME and KONGA\_DB\_PASSWORD.

docker-compose.yml  

    version: '3.7'
    services:
        db:
            build:
              context: postgres
            environment:
              POSTGRES_USERS: ${KONG_DB_USERNAME}:${KONG_DB_PASSWORD}|${KONGA_DB_USERNAME}:${KONGA_DB_PASSWORD}
              POSTGRES_DATABASES: ${KONG_DB_NAME}:${KONG_DB_USERNAME}|${KONGA_DB_NAME}:${KONGA_DB_USERNAME}
            volumes:
            - persist_volume:/var/lib/postgresql/data
            networks:
            - kong-net
    
        kong-migrations:
          image: kong:latest
          entrypoint: sh -c "sleep 10 && kong migrations bootstrap -v"
          environment:
            KONG_DATABASE: ${KONG_DATABASE}
            KONG_PG_HOST: ${KONG_DB_HOST}
            KONG_PG_DATABASE: ${KONG_DB_NAME}
            KONG_PG_USER: ${KONG_DB_USERNAME}
            KONG_PG_PASSWORD: ${KONG_DB_PASSWORD}
          depends_on:
          - db
          networks:
          - kong-net
          restart: on-failure
    

Enter fullscreen mode Exit fullscreen mode

Before we run the actual kong we need to run kong migrations command first. Also if you're wondering why we add sleep on entrypoint it's because i encounter some issue before when i tried to follow tutorial from [kong](https://github.com/Kong/docker-kong/tree/master/compose). Apparently this is because we tried to run migrations command but postgres container is not ready yet, to make it works i just add delay before executing migrations.

Don't forget to define network and volume at bottom or top part of docker-compose  

    volumes:
      persist_volume:
    
    networks:
      kong-net:
        external: true
    

Enter fullscreen mode Exit fullscreen mode

Before run we need to set required environment by postgres and kong

.env  

    # KONG SETTING
    KONG_DB_NAME=db_kong
    KONG_DB_USERNAME=konguser
    KONG_DB_PASSWORD=kongpassword
    KONG_DB_HOST=db
    KONG_DB_PORT=5432
    
    # KONGA SETTING
    KONGA_DB_NAME=db_konga
    KONGA_DB_USERNAME=kongauser
    KONGA_DB_PASSWORD=kongapassword
    KONGA_DB_HOST=db
    KONGA_DB_PORT=5432
    

Enter fullscreen mode Exit fullscreen mode

Now we can try run both container  

    docker network create kong-net
    docker-compose up -d --build
    

Enter fullscreen mode Exit fullscreen mode

Let's add more stuff, this time we add services called kong.  
docker-compose.yml  

        kong:
          image: kong:latest
          environment:
            KONG_DATABASE: ${KONG_DATABASE}
            KONG_PG_HOST: ${KONG_DB_HOST}
            KONG_PG_DATABASE: ${KONG_DB_NAME}
            KONG_PG_USER: ${KONG_DB_USERNAME}
            KONG_PG_PASSWORD: ${KONG_DB_PASSWORD}
            KONG_PROXY_ACCESS_LOG: ${KONG_PROXY_ACCESS_LOG}
            KONG_ADMIN_ACCESS_LOG: ${KONG_ADMIN_ACCESS_LOG}
            KONG_PROXY_ERROR_LOG: ${KONG_PROXY_ERROR_LOG}
            KONG_ADMIN_ERROR_LOG: ${KONG_ADMIN_ERROR_LOG}
            #KONG_ADMIN_LISTEN: ${KONG_ADMIN_LISTEN}
          restart: on-failure
          ports:
          - $KONG_PROXY_PORT:8000
          - $KONG_PROXY_SSL_PORT:8443
            #- $KONG_PROXY_ADMIN_API_PORT:8001
            #- $KONG_PROXY_ADMIN_SSL_API_PORT:8444
          networks:
          - kong-net
    

Enter fullscreen mode Exit fullscreen mode

Don't forget to add more variable to our environment variable  
.env  

    KONG_DATABASE=postgres
    KONG_PROXY_ACCESS_LOG=/dev/stdout
    KONG_ADMIN_ACCESS_LOG=/dev/stdout
    KONG_PROXY_ERROR_LOG=/dev/stderr
    KONG_ADMIN_ERROR_LOG=/dev/stderr
    KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl
    
    KONG_PROXY_PORT=8000
    KONG_PROXY_SSL_PORT=8443
    KONG_PROXY_ADMIN_API_PORT=8001
    KONG_PROXY_ADMIN_SSL_API_PORT=8444
    

Enter fullscreen mode Exit fullscreen mode

Now let's run the whole compose (Postgres + Kong migrations + Kong)  

    docker-compose up -d --build
    

Enter fullscreen mode Exit fullscreen mode

Let's check our kong  

    curl -i http://localhost:8001/
    

Enter fullscreen mode Exit fullscreen mode

You should get some JSON response.

[](#konga)Konga
---------------

*   [Add Konga to Compose](#add-konga-to-docker-compose)
*   [Setup Kong LoopBack](#setup-kong-loopback)
*   [Enable Key Auth Plugin](#enable-key-auth-plugin)
*   [Add Konga as Consumer](#add-konga-as-consumer)
*   [Create API Key for Konga](#create-api-key-for-konga)
*   [Setup Connection](#setup-connection)

Some people might prefer interact with [Kong Admin API](https://docs.konghq.com/2.0.x/admin-api/) through curl or Postman. But i'm more comfortable interacting through UI. [Konga](https://pantsel.github.io/konga/) basically done this, they provide GUI for interacting with Kong Admin API.

### [](#add-konga-to-docker-compose)Add Konga to Docker Compose

To setup Konga with Kong we need few more step, first we need to add Konga in our docker-compose and later we need to configure Kong Admin LoopBack.

Let's do the first step, add konga to our docker-compose.yml  

        konga:
          image: pantsel/konga
          environment:
            TOKEN_SECRET: ${KONGA_TOKEN_SECRET}
            DB_ADAPTER: ${KONG_DATABASE}
            DB_HOST: ${KONGA_DB_HOST}
            DB_PORT: ${KONGA_DB_PORT}
            DB_DATABASE: ${KONGA_DB_NAME}
            DB_USER: ${KONGA_DB_USERNAME}
            DB_PASSWORD: ${KONGA_DB_PASSWORD}
            NODE_ENV: ${KONGA_ENV}
            KONGA_HOOK_TIMEOUT: 10000
          restart: on-failure
          ports:
          - $KONGA_PORT:1337
          depends_on:
          - db
          networks:
          - kong-net
    

Enter fullscreen mode Exit fullscreen mode

And also don't forget to add more variable to .env  

    KONGA_TOKEN_SECRET=some-secret-token
    KONGA_ENV=development
    KONGA_PORT=9000
    

Enter fullscreen mode Exit fullscreen mode

After everything configured now let's run our docker-compose again  

    docker-compose up -d --build
    

Enter fullscreen mode Exit fullscreen mode

Now you can go to your browser and open [http://localhost:9000/](http://localhost:9000/)

Konga would ask you to configure some credentials (Username + Password) that required to access Konga Web. After that they going to prompt how we want to communicate with Kong there's 3 available option:

*   Default (Not Recommended): Konga would access the Kong Admin API directly
*   Key Auth (Recommended): Konga would access Kong Admin API that run behind Kong (Loop-Back API) using configured Api Key.
*   JWT (Recommended): Konga would access Kong Admin API that run behind Kong (Loop-Back API) using JWT by using shared key and secrets.

In this article i would use the second method using Key Auth. Before we setup connection let's go back to access our Kong Admin API. Currently Kong Admin API is publicly accessible for those anyone who has access to the url which is dangerous for production environment. Now we need to protect Kong Admin API by running it behind Kong using [LoopBack](https://docs.konghq.com/2.0.x/secure-admin-api/#kong-api-loopback).

### [](#setup-kong-loopback)Setup Kong LoopBack

Thanks to Kong's routing design it's possible to serve Admin API itself behind Kong proxy.

To configure this we need to following steps:

*   Add Kong Admin API as services: You can import [postman collection](https://github.com/vousmeevoyez/kong-konga-example/blob/master/Kong%20Admin%20API.postman_collection.json) that i already create before or simply copy-paste following commands:

    curl --location --request POST 'http://localhost:8001/services/' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "name": "admin-api",
        "host": "localhost",
        "port": 8001
    }'
    

Enter fullscreen mode Exit fullscreen mode

*   Add Admin API route: To register route on Admin API Services we need either service name or service id, you can replace the following command below:

    curl --location --request POST 'http://localhost:8001/services/{service_id|service_name}/routes' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "paths": ["/admin-api"]
    }'
    

Enter fullscreen mode Exit fullscreen mode

Now our Kong Admin API is running behind Kong Proxy, so in order to access it you need to :  

    curl localhost:8000/admin-api/
    

Enter fullscreen mode Exit fullscreen mode

### [](#enable-key-auth-plugin)Enable Key Auth Plugin

Our Admin API already run behind kong, but is not secured yet. In order to protect Kong Admin API we need to enable key auth plugin at service level by doing this commands.  

    curl -X POST http://localhost:8001/services/{service_id|service_name}/plugins \
        --data "name=key-auth" 
    

Enter fullscreen mode Exit fullscreen mode

### [](#add-konga-as-consumer)Add Konga as Consumer

Our Admin API already run behind kong, but is not secured yet. In order to protect Kong Admin API we need to enable key auth plugin at service level by doing this commands.  

    curl --location --request POST 'http://localhost:8001/consumers/' \
    --form 'username=konga' \
    --form 'custom_id=cebd360d-3de6-4f8f-81b2-31575fe9846a'
    

Enter fullscreen mode Exit fullscreen mode

### [](#create-api-key-for-konga)Create API Key for Konga

Using Consumer ID that generated when we're adding consumer, we will use that Consumer ID and generate API Key.  

    curl --location --request POST 'http://localhost:8001/consumers/e7b420e2-f200-40d0-9d1a-a0df359da56e/key-auth'
    

Enter fullscreen mode Exit fullscreen mode

### [](#setup-connection)Setup Connection

Now we're already have all required component to setup Konga connection  
[![Konga Connection](https://res.cloudinary.com/practicaldev/image/fetch/s--44ce7wC0--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://raw.githubusercontent.com/vousmeevoyez/kong-konga-example/master/setup.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--44ce7wC0--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://raw.githubusercontent.com/vousmeevoyez/kong-konga-example/master/setup.png)  
Remember in picture above, because Kong and Konga are in the same network they can simply reach each other by using container name.

[](#repository)Repository
-------------------------

For those whose want simply clone it you can access this [repository](https://github.com/vousmeevoyez/kong-konga-example)

[](#faq)FAQ
-----------

For those who encounter error below  

    Failed to seed User Error (E_UNKNOWN) :: Encountered an unexpected error
    error: relation "public.konga_users" does not exist
    

Enter fullscreen mode Exit fullscreen mode

this is due to failed migration of konga container, to fix this, run following instruction below:

*   Stop all containers

    docker-compose down
    docker system prune --volumes ## optional!
    

Enter fullscreen mode Exit fullscreen mode

*   Modify .env and look for KONGA\_ENV

    KONGA_ENV=development
    

Enter fullscreen mode Exit fullscreen mode

If it's configured using production, Konga will not perform db migrations which the cause of the error above. So development should work and will trigger user migration for Konga. Now you should able to access it on localhost:9000 if you still use the default configuration.  
[![konga-login](https://res.cloudinary.com/practicaldev/image/fetch/s--S2ELWzZK--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://raw.githubusercontent.com/vousmeevoyez/kong-konga-example/master/Screen%2520Shot%25202020-12-03%2520at%252007.28.18.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--S2ELWzZK--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://raw.githubusercontent.com/vousmeevoyez/kong-konga-example/master/Screen%2520Shot%25202020-12-03%2520at%252007.28.18.png)
