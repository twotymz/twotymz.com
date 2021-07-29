---
layout: post
title: Cloning supbase.io for Local Development
date: 2021-07-28 00:00:00 -0000
excerpt: "Our team recently needed the ability to spin up a [supabase.io](http://supabase.io) back end for local development.  After some trial and error I came up with a working solution mostly based on the Supabase [repo](https://github.com/supabase/supabase) but with some quality of life modifications and removing services we didn't use."
---
Our team recently needed the ability to spin up a [supabase.io](http://supabase.io) back end for local development.  After some trial and error I came up with a working solution mostly based on the Supabase [repo](https://github.com/supabase/supabase) but with some quality of life modifications and removing services we didn't use.

# My Goals

I set out to get all the underlying Supabase services running on the ports of my choosing.  Also, make no modifications to our front-end code other than modifying environment variables so the [supabase-js client](https://github.com/supabase/supabase-js) continues to work as expected.

# TLDR

If you want to skip all the details or follow along as you read the rest, I created a template repo [here](https://github.com/twotymz/supabase_clone) that contains everything I ended up doing.

# Why I Didn't Just Use Supabase's Repo

I started first by cloning the [official Supabase repo](https://github.com/supabase/supabase).  I was hoping I could just modify a couple `.env` files and `docker-compose up` and be done. That wasn't the case. The main issue I ran in to was that I couldn't modify the ports the services were mapped to on localhost. It turns out that's not what the variables in their `.env` configure.  I looked into their `docker-compose.yml` file and basically they expect things to work on very specific ports and that's fine it just doesn't work for us.  A second minor issue was that there was hard coded keys used to initialize the `kong` container and I didn't really like that. I thought it'd be cooler if those could come from the Docker `.env`.

After spending some time looking through their `docker-compose.yml` file and the other Dockerfiles for `kong` and `postgres` I realized that I could probably just copy their setup for the most part in our project and make the changes necessary so it worked as I wanted. This basically entailed coming up with a `docker-compose.yml` file that:

- starts the `kong` service and configures routes to the other services
- starts `PostgreSQL` and initializes the database with Supabase's schemas
- starts the rest of the Supabase services: `GoTrue`, `PostgREST`, and `supabase realtime`

# The Kong Service

Supabase builds the `kong` service locally from the official `kong:2.1` Docker image.  They do this so they can provide the image a custom `kong.yml` file that sets up the routing to the Supabase services.  I figured I'd do the same thing.  I started by copying the `Dockerfile` and the `kong.yml` file from [their](https://github.com/supabase/supabase/tree/master/docker/dockerfiles/kong) repo into my working directory at `/kong`. The `kong.yml` file from supabase had the anon and service-role keys hard coded in it so I modified the file so that keys could be passed in at build time.  I did this by replacing the keys with unique strings that would be easy to find and replace later.  I also removed the `storage` service because it's not a feature we needed for our project.

Here's the updated `kong.yml` file:

```yaml
_format_version: "1.1"
services:
-   name: auth-v1-open
    url: http://auth:9999/verify
    routes:
    -   name: auth-v1-open
        strip_path: true
        paths:
        - /auth/v1/verify
    plugins:
    -   name: cors
-   name: auth-v1-open-callback
    url: http://auth:9999/callback
    routes:
    -   name: auth-v1-open-callback
        strip_path: true
        paths:
        - /auth/v1/callback
    plugins:
    -   name: cors
-   name: auth-v1-open-authorize
    url: http://auth:9999/authorize
    routes:
    -   name: auth-v1-open-authorize
        strip_path: true
        paths:
        - /auth/v1/authorize
    plugins:
    -   name: cors
-   name: auth-v1
    _comment: 'GoTrue: /auth/v1/* -> http://auth:9999/*'
    url: http://auth:9999/
    routes:
    -   name: auth-v1-all
        strip_path: true
        paths:
        - /auth/v1/
    plugins:
    -   name: cors
    -   name: key-auth
        config:
            hide_credentials: true
-   name: rest-v1
    _comment: 'PostgREST: /rest/v1/* -> http://rest:3000/*'
    url: http://rest:3000/
    routes:
    -   name: rest-v1-all
        strip_path: true
        paths:
        - /rest/v1/
    plugins:
    -   name: cors
    -   name: key-auth
        config:
            hide_credentials: true
-   name: realtime-v1
    _comment: 'Realtime: /realtime/v1/* -> ws://realtime:4000/socket/*'
    url: http://realtime:4000/socket/
    routes:
    -   name: realtime-v1-all
        strip_path: true
        paths:
        - /realtime/v1/
    plugins:
    -   name: cors
    -   name: key-auth
        config:
            hide_credentials: true
consumers:
-   username: 'anon-key'
    keyauth_credentials:
    -   key: SUPABASE_ANON_KEY
-   username: 'service-key'
    keyauth_credentials:
    -   key: SUPABASE_SERVICE_ROLE_KEY
```

Next, I modified the `Dockerfile` so that I could accept the anon key and service role keys at build time via Docker build args.  Then I added some `sed` commands to replace the strings `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY` in `kong.yml` after it was copied to the image with the values from the corresponding build args.

Here's my updated `Dockerfile`:

```dockerfile
FROM kong:2.1

# Build time defaults
ARG build_KONG_DATABASE=off
ARG build_KONG_PLUGINS=request-transformer,cors,key-auth
ARG build_KONG_DECLARATIVE_CONFIG=/var/lib/kong/kong.yml
ARG build_SUPABASE_ANON_KEY
ARG build_SUPABASE_SERVICE_ROLE_KEY

# Run time values
ENV KONG_DATABASE=$build_KONG_DATABASE
ENV KONG_PLUGINS=$build_KONG_PLUGINS
ENV KONG_DECLARATIVE_CONFIG=$build_KONG_DECLARATIVE_CONFIG

USER root
COPY kong.yml /var/lib/kong/kong.yml
RUN sed -i "s/SUPABASE_ANON_KEY/$build_SUPABASE_ANON_KEY/g" /var/lib/kong/kong.yml
RUN sed -i "s/SUPABASE_SERVICE_ROLE_KEY/$build_SUPABASE_SERVICE_ROLE_KEY/g" /var/lib/kong/kong.yml
```

# The PostgreSQL Service

The `PostgreSQL` service is handled in basically the same manner by Supabase as the `kong` service.  They build it locally from their own `supabase/postgres` image which is "unmodified Postgres with some useful plugins" and they throw in some custom SQL initialization scripts that, from what I can tell, add the `auth` schema for the `GoTrue` service and sets up the database roles and users. I copied over all the files from [here](https://github.com/supabase/supabase/tree/master/docker/dockerfiles/postgres) into a `/postgres` directory in my local working directory and started making my changes.

The first thing I did was collect all the SQL files into a directory called `init.d`.  I then deleted the `storage-schema.sql` file because storage is not a service we care about. I then renamed `auth-schema.sql` to `10-auth-schema.sql` and added another file called `15-post-auth-schema.sql`.  Here's what that file looked like:

```postgres
ALTER ROLE postgres SET search_path = "$user", auth, public, extensions;
```

`15-post-auth-schema.sql` was added as more of a convenience to make extensions available to other SQL scripts we run after the Supabase scripts initialize the database.  Renaming the files so that they are prefixed by a number was to control the order in which the files would be run.  What we do in our project is add our own SQL files that create our project specific database tables and populate them with our own test data.  There are several of these files and they all start with a number after `15`.

Next I modified the `Dockerfile` to just copy the entire `init.d` directory into the image at `/docker-entrypoint-inidb.d` and I got rid of most of the other stuff because it just seemed redundant to me.  Here's the `Dockerfile` I ended up with.

```dockerfile
FROM supabase/postgres:0.13.0
COPY /init.d /docker-entrypoint-initdb.d
```

# The Rest of the Supabase Services

`kong` and `PostgreSQL` were the only services that required local configuration files to be present in the images. `GoTrue`, `PostgREST`, and `supabase realtime` are all configured via environment variables in the `docker-compose.yml` file.  So my next task was to take their `docker-compose.yml` file and modify it.  My goals here were to:

1. take advantage of the `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY` build args so the values could come from `.env` to build the `kong` image correctly.
2. map service ports to [localhost](http://localhost) based on the settings in `.env` but make sure all the services can reach each other inside the containers.
3. remove the `storage` service.

Number 1 and 3 were easy. Number 2 wasn't that bad.  It mainly entailed exposing the services on their default ports inside the containers, mapping the ports externally to my defined ports, and fixing up the database connection strings.

Here's what I ended up with:

```yaml
version: "2.0"

services:

  kong:
    build:
      context: ./kong
      args:
        build_SUPABASE_ANON_KEY: ${SUPABASE_ANON_KEY}
        build_SUPABASE_SERVICE_ROLE_KEY: ${SUPABASE_SERVICE_ROLE_KEY}
    environment:
      KONG_DECLARATIVE_CONFIG: /var/lib/kong/kong.yml
      KONG_PLUGINS: request-transformer,cors,key-auth,http-log
    ports:
      - ${KONG_PORT}:8000
      - ${KONG_PORT_TLS}:8443
    expose:
      - 8000
      - 8443

  auth:
    image: supabase/gotrue:latest
    ports:
      - ${AUTH_PORT}:9999
    depends_on:
      - db
    restart: always
    environment:
      GOTRUE_JWT_SECRET: ${JWT_SECRET}
      GOTRUE_JWT_EXP: 3600
      GOTRUE_JWT_DEFAULT_GROUP_NAME: authenticated
      GOTRUE_DB_DRIVER: postgres
      GOTRUE_API_HOST: 0.0.0.0
      PORT: 9999
      GOTRUE_SITE_URL: localhost
      GOTRUE_MAILER_AUTOCONFIRM: 'true'
      GOTRUE_LOG_LEVEL: DEBUG
      GOTRUE_OPERATOR_TOKEN: ${OPERATOR_TOKEN}
      DATABASE_URL: 'postgres://postgres:${POSTGRES_PASSWORD}@db:5432/postgres?sslmode=disable'
    expose:
      - 9999

  rest:
    image: postgrest/postgrest:nightly-2021-03-05-19-03-d3a8b5f
    ports:
      - ${REST_PORT}:3000
    depends_on:
      - db
    restart: always
    environment:
      PGRST_DB_URI: postgres://postgres:${POSTGRES_PASSWORD}@db:5432/postgres
      PGRST_DB_SCHEMA: public
      PGRST_DB_ANON_ROLE: anon
      PGRST_JWT_SECRET: ${JWT_SECRET}
    expose:
      - 3000

  realtime:
    image: supabase/realtime:latest
    ports:
      - ${REALTIME_PORT}:4000
    depends_on:
      - db
    restart: on-failure
    environment:
      DB_HOST: db
      DB_NAME: postgres
      DB_USER: postgres
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      DB_PORT: 5432
      PORT: 4000
      HOSTNAME: localhost
      # Disable JWT Auth locally. The JWT_SECRET will be ignored.
      SECURE_CHANNELS: 'false'
      JWT_SECRET: ${JWT_SECRET}
    expose:
      - 4000

  db:
    image: my_db:latest
    build:
      context: ./postgres
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    ports:
      - ${POSTGRES_PORT}:5432
    expose:
      - 5432
```

# The Docker .env

The final piece was a Docker `.env` file that allows the developer to specify the ports, passwords, and secrets.

```bash
POSTGRES_PASSWORD=your-super-secret-db-password

# If you change JWT_SECRET you'd have to generate new keys for
# SUPABASE_ANON_KEY and SUPABASE_SERVICE_ROLE.
JWT_SECRET=d05855fb-a082-47e6-986c-8826c6f8dbf2

SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYW5vbiIsImlhdCI6MTYyNDM4MDYzNCwiZXhwIjoxOTM5OTU2NjM0fQ.SDH7wXuJ0WMRpgvSLIolzqI8wn7XAdl8p5niE8y-PYw
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoic2VydmljZV9yb2xlIiwiaWF0IjoxNjI0MzgwNjM0LCJleHAiOjE5Mzk5NTY2MzR9.DXrT8gTevzIEBTJwWLwEDc1VzyzOe5Y3EggHAU8TNBE

# Port mapping for services
POSTGRES_PORT=60001
AUTH_PORT=60002
REST_PORT=60003
REALTIME_PORT=60004
KONG_PORT=60005
KONG_PORT_TLS=60006
```

*Don't worry. I generated those keys specifically for the template repo. They aren't used in production.*

With this `.env` file we have:

- `PostgreSQL` accessible at `[localhost:60001](http://localhost:60001)` The username is `postgres` and the password is `your-super-secret-db-password`.
- `GoTrue` is accessible at `localhost:60002`.
- `PostgREST` is accessible at `localhost:60003`.
- `supabase realtime` service is accessible at `localhost:60004`.
- Kong is accessible at `localhost:60005` and is what we need to pass in to `createClient` when creating a supabase JS client instance.

# Future Considerations

I'm not super excited that this solution requires us keeping our project up-to-date with changes to the Supabase repository which is in active development. I'm not sure how much of a maintenance problem this is going to be.  Given the alternatives, I'm fine with it for now.  We're really hoping the Supabase team eventually provides an official template repository that we could use that provides more freedom in how the back-end gets configured locally.

The `GoTrue` service is configured to automatically confirm sign-ups in this configuration. It might be nice to allow the developer to configure the `GoTrue` container to use `SMTP` in development via the `.env` file.

# Summary

So far this seems to be working fairly well. In our Next JS front-end we can continue to use the Supabase JS client to talk to all the Supabase services locally with zero changes other than updating environment variables. Our server-side client is created like this:

```js
export const supabaseAdmin = createClient(
	process.env.NEXT_PUBLIC_SUPABASE_URL,
	process.env.SUPABASE_SERVICE_ROLE_KEY
)
```

and our client-side client is created like this:

```js
export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
)
```

and this is our `.env.local` for our front end:

```js
SUPABASE_SERVICE_ROLE_KEY="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoic2VydmljZV9yb2xlIiwiaWF0IjoxNjI0MzgwNjM0LCJleHAiOjE5Mzk5NTY2MzR9.DXrT8gTevzIEBTJwWLwEDc1VzyzOe5Y3EggHAU8TNBE"
NEXT_PUBLIC_SUPABASE_ANON_KEY="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYW5vbiIsImlhdCI6MTYyNDM4MDYzNCwiZXhwIjoxOTM5OTU2NjM0fQ.SDH7wXuJ0WMRpgvSLIolzqI8wn7XAdl8p5niE8y-PYw"
NEXT_PUBLIC_SUPABASE_URL="http://localhost:60005"
```

This was a fun project. I don't spend a lot of time keeping up with all these different micro-services and how they can be cobbled together so it was fun getting some hands on experience doing that by seeing how Supabase did it.  I can definitely see myself using `kong`, `PostgREST`, and `GoTrue` for other projects in the future.
