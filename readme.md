[![Kelvin](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](/vousmeevoyez)

[Kelvin](/vousmeevoyez)

Posted on Apr 9, 2020 • Updated on Jul 31, 2021

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

[](#support)Support
-------------------

If you find my articles or project is useful please support me through:

[![Buy Me A Coffee](https://res.cloudinary.com/practicaldev/image/fetch/s--rJvAnNew--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://cdn.buymeacoffee.com/buttons/default-black.png)](https://www.buymeacoffee.com/vousmeevoyez)

Thank you very much

[](#api-gateway-series)Api Gateway series
-----------------------------------------

if you like to know other series please check out below:  
\*[Introduction](https://dev.to/vousmeevoyez/introduction-api-gateway-part-1-3n66)  
\*[Setup Kong](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#api-gateway-series) (This Articles)

Top comments (18)
-----------------

Crown

### Sort discussion:

*   [
    
    Selected Sort Option Top
    
    Most upvoted and relevant comments will be first
    
    ](/vousmeevoyez/setup-kong-konga-part-2-dan?comments_sort=top#toggle-comments-sort-dropdown)
*   [
    
    Latest
    
    Most recent comments will be first
    
    ](/vousmeevoyez/setup-kong-konga-part-2-dan?comments_sort=latest#toggle-comments-sort-dropdown)
*   [
    
    Oldest
    
    The oldest comments will be first
    
    ](/vousmeevoyez/setup-kong-konga-part-2-dan?comments_sort=oldest#toggle-comments-sort-dropdown)

Subscribe

    ![pic](https://res.cloudinary.com/practicaldev/image/fetch/s--RmY55OKL--/c_limit,f_auto,fl_progressive,q_auto,w_256/https://practicaldev-herokuapp-com.freetls.fastly.net/assets/devlogo-pwa-512.png)

Personal Trusted User

![loading](https://dev.to/assets/loading-ellipsis-b714cf681fd66c853ff6f03dd161b77aa3c80e03cdc06f478b695f42770421e9.svg)

[Create template](/settings/response-templates)

Templates let you quickly answer FAQs or store snippets for re-use.

Submit Preview [Dismiss](/404.html)

Collapse Expand

 

[![jamallmahmoudi profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--tHRvKrX3--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/349567/8eaab36f-f152-4774-91a5-fffe383a4dd4.jpeg)](https://dev.to/jamallmahmoudi)

[jamallmahmoudi](https://dev.to/jamallmahmoudi)

jamallmahmoudi

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--KYyuaaDq--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/349567/8eaab36f-f152-4774-91a5-fffe383a4dd4.jpeg)jamallmahmoudi](/jamallmahmoudi)

Follow

*   Location
    
    iran -tehran
    
*   Work
    
    developer and QA at tehran
    
*   Joined
    
    Mar 12, 2020
    

• [Oct 24](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-22ee7)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-22ee7)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/jamallmahmoudi/comment/22ee7)

Hi cool& perfect

Like comment: Like comment: 2 likes Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/22ee7)

Collapse Expand

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Oct 25](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-22f7i)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-22f7i)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/22f7i)

if this helps please support me so i can produce more quality content to help other developers  
[![Buy Me A Coffee](https://res.cloudinary.com/practicaldev/image/fetch/s--rJvAnNew--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://cdn.buymeacoffee.com/buttons/default-black.png)](https://www.buymeacoffee.com/vousmeevoyez)

Like comment: Like comment: 1 like Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/22f7i)

Collapse Expand

 

[![ayyappa99 profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--7G_G7sWh--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/337545/e5f6b4b4-2267-4dd6-bbb5-f0159027367e.png)](https://dev.to/ayyappa99)

[Ayyappa](https://dev.to/ayyappa99)

Ayyappa

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--1GeTbrOa--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/337545/e5f6b4b4-2267-4dd6-bbb5-f0159027367e.png)Ayyappa](/ayyappa99)

Follow

*   Location
    
    Hyderabad
    
*   Work
    
    Game Developer
    
*   Joined
    
    Feb 18, 2020
    

• [Jul 22 '20](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-12bog)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-12bog)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/ayyappa99/comment/12bog)

[@vousmeevoyez](https://dev.to/vousmeevoyez) : Unfortunately, running the same exact script giving below error  
'FATAL: database "konguser" does not exist'

Like comment: Like comment: 1 like Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/12bog)

Collapse Expand

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Jul 23 '20](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-12d75)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-12d75)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/12d75)

Hi could you share more details? do you run it using start.sh? or just docker-compose?

Like comment: Like comment: 1 like Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/12d75)

Collapse Expand

 

[![giesberge profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dwvC4Ecn--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/677095/6ac2ca80-cbb5-43b9-915c-e61cd672e942.png)](https://dev.to/giesberge)

[giesberge](https://dev.to/giesberge)

giesberge

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--1hlNIvj4--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/677095/6ac2ca80-cbb5-43b9-915c-e61cd672e942.png)giesberge](/giesberge)

Follow

*   Joined
    
    Jul 30, 2021
    

• [Jul 30 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gm3f)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gm3f)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/giesberge/comment/1gm3f)

I've tried from both and I have the same error,

Like comment: Like comment: 2 likes Like Thread Thread

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Jul 31 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gmbp)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gmbp)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/1gmbp)

hi could you share your error?

Like comment: Like comment: 2 likes Like Thread Thread

 

[![giesberge profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dwvC4Ecn--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/677095/6ac2ca80-cbb5-43b9-915c-e61cd672e942.png)](https://dev.to/giesberge)

[giesberge](https://dev.to/giesberge)

giesberge

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--1hlNIvj4--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/677095/6ac2ca80-cbb5-43b9-915c-e61cd672e942.png)giesberge](/giesberge)

Follow

*   Joined
    
    Jul 30, 2021
    

• [Jul 31 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gmj4)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gmj4)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/giesberge/comment/1gmj4)

database "konguser" does not exist. This is also after running a docker system prune:  
[pastebin.com/yJkp7BYK](https://pastebin.com/yJkp7BYK)

Like comment: Like comment: 1 like Like Thread Thread

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Aug 1 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gn8n)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gn8n)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/1gn8n)

Hi giesberge can you try with this following sequence

*   docker system prune --volumes
*   mv .env.sample .env
*   update .env from

    KONGA_ENV=production
    

Enter fullscreen mode Exit fullscreen mode

to  

    KONGA_ENV=development
    

Enter fullscreen mode Exit fullscreen mode

*   ./start.sh
*   go to localhost:9000

if this helps please support me so i can produce more quality content to help other developers  
[![Buy Me A Coffee](https://res.cloudinary.com/practicaldev/image/fetch/s--rJvAnNew--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://cdn.buymeacoffee.com/buttons/default-black.png)](https://www.buymeacoffee.com/vousmeevoyez)

Like comment: Like comment: 1 like Like Thread Thread

 

[![giesberge profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dwvC4Ecn--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/677095/6ac2ca80-cbb5-43b9-915c-e61cd672e942.png)](https://dev.to/giesberge)

[giesberge](https://dev.to/giesberge)

giesberge

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--1hlNIvj4--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/677095/6ac2ca80-cbb5-43b9-915c-e61cd672e942.png)giesberge](/giesberge)

Follow

*   Joined
    
    Jul 30, 2021
    

• [Aug 3 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1go81)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1go81)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/giesberge/comment/1go81)

Already done, I even tried changing the db\_names but that didn't work. [pastebin.com/wmcHb9Tw](https://pastebin.com/wmcHb9Tw)

Like comment: Like comment: 1 like Like Thread Thread

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Aug 3 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gob3)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1gob3)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/1gob3)

I've notice you're using windows. Did you use wsl? And what terminal are you running? Command prompt?

Like comment: Like comment: 1 like Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/1gob3)

Collapse Expand

 

[![ayyappa99 profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--7G_G7sWh--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/337545/e5f6b4b4-2267-4dd6-bbb5-f0159027367e.png)](https://dev.to/ayyappa99)

[Ayyappa](https://dev.to/ayyappa99)

Ayyappa

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--1GeTbrOa--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/337545/e5f6b4b4-2267-4dd6-bbb5-f0159027367e.png)Ayyappa](/ayyappa99)

Follow

*   Location
    
    Hyderabad
    
*   Work
    
    Game Developer
    
*   Joined
    
    Feb 18, 2020
    

• [Jul 23 '20](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-12do7)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-12do7)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/ayyappa99/comment/12do7)

I ran with start.sh script  
Also what i noticed was the db name needs to be kong always else kong is throwing kong database doesn't exist.

Like comment: Like comment: 2 likes Like Thread Thread

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Aug 30 '20](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-148f4)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-148f4)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/148f4)

yes, I forgot to mention that in article. thanks man

Like comment: Like comment: 2 likes Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/148f4)

Collapse Expand

 

[![tiagosmx profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--PT5PyjIT--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/592260/ea44807f-eabc-4e06-ac41-2be98a58284d.jpeg)](https://dev.to/tiagosmx)

[Tiago Stapenhorst Martins](https://dev.to/tiagosmx)

Tiago Stapenhorst Martins

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--xLZc_nh4--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/592260/ea44807f-eabc-4e06-ac41-2be98a58284d.jpeg)Tiago Stapenhorst Martins](/tiagosmx)

Follow

*   Location
    
    Curitiba, Brazil
    
*   Work
    
    Senior Backend Engineer at Ubivis
    
*   Joined
    
    Mar 8, 2021
    

• [Mar 8 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1c822)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1c822)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/tiagosmx/comment/1c822)

Brilliant post dude! Your guide is valuable and well written.  
I have been trying with no success to setup Kong and Konga with docker a while ago by following their git and official pages. Their documentation aren't very friendly indeed.  
Thanks!

Like comment: Like comment: 1 like Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/1c822)

Collapse Expand

 

[![tikam02 profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--KY8c_ywU--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/450046/7913117f-ea07-46a9-935e-514a2cadb972.png)](https://dev.to/tikam02)

[Tikam Singh Alma](https://dev.to/tikam02)

Tikam Singh Alma

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--trFgQtA4--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/450046/7913117f-ea07-46a9-935e-514a2cadb972.png)Tikam Singh Alma](/tikam02)

Follow

Design | Debug | Develop

*   Email
    
    [tikamalma1@gmail.com](mailto:tikamalma1@gmail.com)
    
*   Location
    
    NYC,USA
    
*   Education
    
    DAIICT
    
*   Work
    
    HuddleUp
    
*   Joined
    
    Aug 10, 2020
    

• [Aug 10 '20](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-13ckc)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-13ckc)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/tikam02/comment/13ckc)

I ran with start.sh script  
db\_1 | FATAL: database "konguser" does not exist

Like comment: Like comment: 2 likes Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/13ckc)

Collapse Expand

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Aug 30 '20](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-148f3)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-148f3)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/148f3)

Hi Tikam, sorry for slow response. Been very busy this couple months. try stop and clearing all volumes  

    docker-compose down
    docker system prune --volumes
    

Enter fullscreen mode Exit fullscreen mode

also use this setting as .env  

    # KONG SETTING
    KONG_DB_NAME=db_kong
    KONG_DB_USERNAME=konguser
    KONG_DB_PASSWORD=kongpassword
    KONG_DB_HOST=db
    KONG_DB_PORT=5432
    
    KONG_DATABASE=postgres
    KONG_PROXY_ACCESS_LOG=/dev/stdout
    KONG_ADMIN_ACCESS_LOG=/dev/stdout
    KONG_PROXY_ERROR_LOG=/dev/stderr
    KONG_ADMIN_ERROR_LOG=/dev/stderr
    KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl
    
    KONG_PROXY_PORT=8000
    KONG_PROXY_SSL_PORT=8443
    
    # KONGA SETTING
    KONGA_DB_NAME=db_konga
    KONGA_DB_USERNAME=kongauser
    KONGA_DB_PASSWORD=kongapassword
    KONGA_DB_HOST=db
    KONGA_DB_PORT=5432
    
    KONGA_TOKEN_SECRET=some-secret-token
    KONGA_ENV=production
    KONGA_PORT=9000
    

Enter fullscreen mode Exit fullscreen mode

Like comment: Like comment: 6 likes Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/148f3)

Collapse Expand

 

[![tikam02 profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--KY8c_ywU--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/450046/7913117f-ea07-46a9-935e-514a2cadb972.png)](https://dev.to/tikam02)

[Tikam Singh Alma](https://dev.to/tikam02)

Tikam Singh Alma

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--trFgQtA4--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/450046/7913117f-ea07-46a9-935e-514a2cadb972.png)Tikam Singh Alma](/tikam02)

Follow

Design | Debug | Develop

*   Email
    
    [tikamalma1@gmail.com](mailto:tikamalma1@gmail.com)
    
*   Location
    
    NYC,USA
    
*   Education
    
    DAIICT
    
*   Work
    
    HuddleUp
    
*   Joined
    
    Aug 10, 2020
    

• [Aug 30 '20](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-148go)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-148go)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/tikam02/comment/148go)

Thank you so much @kelvin for the help.

Like comment: Like comment: 2 likes Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/148go)

Collapse Expand

 

[![amirhs712 profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--ncGSNocr--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/350398/7daaf323-9fea-4921-901c-9a319b45613c.png)](https://dev.to/amirhs712)

[amirhs712](https://dev.to/amirhs712)

amirhs712

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--FFOGhcbk--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/350398/7daaf323-9fea-4921-901c-9a319b45613c.png)amirhs712](/amirhs712)

Follow

*   Joined
    
    Mar 14, 2020
    

• [Sep 29 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1idno)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1idno)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/amirhs712/comment/1idno)

where do I get docker-entrypoint.sh?

Like comment: Like comment: 1 like Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/1idno)

Collapse Expand

 

[![vousmeevoyez profile image](https://res.cloudinary.com/practicaldev/image/fetch/s--dlxSxC6u--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)](https://dev.to/vousmeevoyez)

[Kelvin Author](https://dev.to/vousmeevoyez)

Kelvin

 [![](https://res.cloudinary.com/practicaldev/image/fetch/s--i2zzVa24--/c_fill,f_auto,fl_progressive,h_90,q_auto,w_90/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/353129/e4043302-8b64-4731-8fd5-06d4e8d7c120.jpg)Kelvin](/vousmeevoyez)

Follow

Tech Wizard (Software Engineer + DevOps + QA + AI) | Startup Entrepreneur

*   Email
    
    [kelvindsmn@gmail.com](mailto:kelvindsmn@gmail.com)
    
*   Location
    
    Indonesia
    
*   Education
    
    Informatic Engineering
    
*   Work
    
    DevOps Engineer
    
*   Joined
    
    Mar 20, 2020
    

Author

• [Oct 4 '21](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1ihf6)

Dropdown menu

*   [Copy link](https://dev.to/vousmeevoyez/setup-kong-konga-part-2-dan#comment-1ihf6)

*   Hide

*   [Report abuse](/report-abuse?url=https://dev.to/vousmeevoyez/comment/1ihf6)

Hi amirhs, terribly sorry for slow response. You can find the docker-entrypoint for the postgres container here [github.com/vousmeevoyez/kong-konga...](https://github.com/vousmeevoyez/kong-konga-example/blob/master/postgres/docker-entrypoint.sh)

Like comment: Like comment: 1 like Like [Comment button Reply](#/vousmeevoyez/setup-kong-konga-part-2-dan/comments/new/1ihf6)

[View full discussion (18 comments)](/vousmeevoyez/setup-kong-konga-part-2-dan/comments)

[Code of Conduct](/code-of-conduct) • [Report abuse](/report-abuse)

Are you sure you want to hide this comment? It will become hidden in your post, but will still be visible via the comment's [permalink](#).

Hide child comments as well

Confirm

For further actions, you may consider blocking this person and/or [reporting abuse](/report-abuse)