# docker-node-pg
Node app in docker container with postgres/ express / react integration

## dockerfile
This is the build step of docker image.  When running `docker build -t docker-node-pg:latest`, docker will run through the docker file run by line as if it is setting up the app and its' dependencies from scratch.  

```
# starting point.  Runs all commands from a node image (from dockerhub)
FROM node:10
# sets the working directory of the app.  If the direcoty does not exist, it will be created.
WORKDIR /app
# copies the package.json file from root of host directory to working directory in docker container
COPY ./package.json .
# runs npm install all (production + development) (verbose to show any errors if any)
RUN npm install --verbose
#copies all files from host root directory to working directory in container.  Copies from prior are cached and will not be duplicated
COPY . .
# runs node server
CMD "npm start"
```

## docker-compose.yml (production)
A list of things we need to know before all of this is configured...
1. What are the credentials for the postgres db (username, password, host, port)?
   - For postgres, we add environment variables `POSTGRES_USER`, `POSTGRES_PASSWORD`, `HOST`, `POSTGRES_PORT`, where...
   - `POSTGRES_USER=postgres` // This can be anything
   - `POSTGRES_PASSWORD=docker` // This can be anything
   - `HOST=db` // This is important... It is the name of the service the postgres docker container is created from
   - `POSTGRES_PORT=5432` // This is by convention
2. What port is the server listening on?
   -  `PORT=3000` // This can be anything, as long as it is not already used on the host.  
3. How do we start the postgres db (createdb, add our credentials)?
   - This is generally done using a start script.  We will get to this later
4. How do we seed the db?
   - This is done on the server docker container.  We would run some `schema.sql` file or `node seed.js` script.  
5. How do we wait for the db to be seeded before we can connect the server to it?
   - This is using a `wait-for-it.sh` file.  This is a provided file for the postgres image, and can be replaced.   
6. How do we connect to the db?
   - This is done through the bridge network.  By default, all services on a `docker-compose.yml` are connected through a bridge network.  This can be changed to a different network type, but for this case, a bridge network will work best.  The way we can connect the web container to the db container is by using `HOST` mentioned before.  By naming the postgres container `db`, we are specifying the host of the postgres container to be `db`.  This hostname can be used in other containers by adding a property on the web container `depends_on` to `db`.  
   
Now that we have all the info we need for this, we can get started on the `docker-compose` file...
```
#Docker-compose.yml (production)
version: '2'
services:
  server:
    build: .
    image: docker-pg
    environment:
      - PORT=3000
      - HOST=db
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=docker
      - POSTGRES_PORT=5432
    depends_on: 
      - db
    ports:
      - "3001:3000"
    command: ["./wait-for-it.sh", "db:5432", "--", "npm", "run", "start"]
  
  db:
    image: postgres
    ports: 
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=docker
    volumes:
      - ./tmp:/docker-entrypoint-initdb.d
```
There is some stuff we didn't mention before, so let's go through what all of this is...

### Properties
- `server` is the service container name
- `build` is the context where the docker image is built.  If `.`, this means root of the directory
- `image` is the image name when it is created
- `environment` is a list of environment variables available to the container, and is automatically provided through `process.env`
- `depends_on` was mentioned before, but is how we link the web container to the db container.  The web container depends on the db.  
- `ports` is how we add port mapping.  This is not to be confused with port exposing.  "3001:3000" means that port 3001 will be used on the host machine where the docker container listens on port 3000.  Exposing is not required unless you want to connect these containers with containers outside of the network.
- `command` is the execution script to start the container.  This will start the app
- `volumes` is how you mount volumes in the docker container or mount volumes from the container to the local machine.

## Postgres

### Starting postgres
Once the `docker-compose` file is created, we can figure out how to start postgres.  We can do this by using a start script.  Within the volumes property of the `db` container, we see `- ./tmp:/docker-entrypoint-initdb.d`.  This is a specific volume inside the postgres container, which is built in.  Anything copied to the directory `/docker-entrypoint-initdb.d` is automatically executed when the container is started.  We mount this volume to a `./tmp` directory on the host machine.  Within the directory is a shell script with the following... 
```
#!/bin/bash

set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE USER docker;
    CREATE DATABASE docker;
    GRANT ALL PRIVILEGES ON DATABASE docker TO docker;
EOSQL
```
This is not a shell scripting tutorial, so I will not go over this.  All this does is create default credentials superUser `docker` (no password required) and database `docker`.  It will also add additional credentials to the postgres db with the credentials provided in the `environment` property of the `docker-compose.yml` file.   

### Seed db
As mentioned above, anything added in the mounted volume to `/docker-entrypoint-initdb.d` will be executed, including `*.sql`, `*.sql.gz`, or `*.sh` will be run.  So we add our `schema.sql` file in the directory.       

```
DROP TABLE IF EXISTS users;

CREATE TABLE users (
    id SERIAL,
    first text,
    last text,
    primary key(id)
);

INSERT into users(first, last) values ('kent', 'ogisu');
INSERT into users(first, last) values ('john', 'doe');
INSERT into users(first, last) values ('jane', 'doe');
```
The database was already created as `docker` so we don't need to create it again nor do we need to check if it was created.  

### Run server after database was created
If you ran the docker container as is, you'd most likely get issues connecting to the db, mainly because the server is running before the postgres docker container is run.    
But wait.... Didn't we specify `depends_on` on the web container, which tells docker to run `db` first???  
yes, that is true, but docker does not wait for `db` to finish running before running the web container `server`.  In order to wait for the db, we need to run a script that loops a timer until the db credentials are set.  

A `wait-for-it.sh` file was added to the root directory of the app.  This file is provided by the `postgres` open source community, and can be copied from [wait-for-it script](https://github.com/vishnubob/wait-for-it).  For more info see [control postgres startup and shutdown](https://docs.docker.com/compose/startup-order/).  
Once the file is copied, an execution command needs to be provided to the server container, which will start the node app when and only when the db is created.  
The execution script is as follows:   
 - `command: ["./wait-for-it.sh", "db:5432", "--", "npm", "run", "start"]`  
This executes the `.sh` file, sets the `host:port`, and then runs node.  



