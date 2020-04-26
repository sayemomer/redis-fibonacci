* **setup docker**

so in we need two dockerfile in our server and worker because we want to ran it as a multi container service . lets began !

first create a docker file at server ,

```FROM node:alpine```

which acctually takes a minimal node.js image from dockerhub ,

```WORKDIR '/app'```

its initialize the working directory inside our contaner ..

COPY ./package.json ./

copy package.json file and give it relative path

```RUN npm install```

npm install command using that copied package file ,

 ```COPY . . ```
 
 copy everything in the working directory , and finally run
 
 ```CMD ["npm" , "run" , "dev"]```
 
 which starts the development server..
 
 so finally the dockerfile on server looks like 
 
 ```
FROM node:alpine
WORKDIR '/app'
COPY ./package.json ./
RUN npm install
COPY . .
CMD ["npm" , "run" , "dev"]
```

so whats happens if I run docker run on this folder .. lets see

run on server folder this

``` docker run -f Dockerfile.dev ```


the -f folder is because we have Dockerfile.dev instead Dockerfile

opps we need to specify the path of that file 

``` docker run -f Dockerfile.dev . ```

after this we have this message in console 

**Successfully built 4e471da5e730**

this is the docker image ID that we will run , so run

``` docker run 4e471da5e730 ```

voila !!

as expected got this

```
Error: connect ECONNREFUSED 127.0.0.1:5432
    at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1141:16) {
  errno: -111,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 5432
}
```

which is accutall valid because we didn't created any configuration for postgress and redis . to solve this issue we have two solution , 1st one is write everything in this Dockerfile.dev and run this image , but we don't want this , we want 
our server and worker run separtae container as a service as well as out redus and postgres .

so to do so we need something called docker-compose , by docker-compose we can create multiple container and create networking between then and run as a service .


so firstly we have two Dockerfile.dev in our worker and server folder why ! as said we want this two run as a service .

so create a ```docker-compose.yml``` file in root 


* **docker compose**


```version: '3'```

specify the version of docker-compose ,

 ```
 services: 
 ```
 
 in this block we will write the services we wanna run as a separte container lets gooo ..
 
 
 so far the file
 
 ```
 version: '3'
services: 
  postgres:
    image: 'postgres:latest'
  redis :
    image: 'redis:latest'
```

postgres and redis service will run as a different service and lastly write our backend server as a build

```
server:
    build: 
    dockerfile: Dockerfile.dev
    context: ./server
```

it will looks for a dockerfile at folder which specified in context , so lets run this file by ```docker-compose up --build```

this time the server is not succesfull connected because of env values but the services which i wrote on compose file
started. so way to go , we need the env varible specify in the compose file , but before that we have another problem .
you can see that we are running nodemon in our server starts in development mode , so that we don't need to restart the
server everytime we make change, 

but in our current configuration its not working . because of our code is running from the app directory in the container , but code changes in local , so we need to reference the code in container .. I will talk about it later.
for now lets make this happen by using volume ,

```
volumes: 
      - /app/node_modules
      - ./server:/app
```

this conf firstly removes the node_moduels in app , then take the reference of everything of server into the app directory of the container . so now server is restarting as expected . lasty we need env to start the server successfully .


```
environment: 
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - PGUSER=postgres
      - PGHOST=postgres
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password
      - PGPORT=5432
```

so finally the compose file looks like this


```

version: '3'
services: 
  postgres:
    image: 'postgres:latest'
  redis :
    image: 'redis:latest'
  server:
    build: 
      dockerfile: Dockerfile.dev
      context: ./server
    volumes: 
      - /app/node_modules
      - ./server:/app
    environment: 
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - PGUSER=postgres
      - PGHOST=postgres
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password
      - PGPORT=5432
```

oh ! wait we didn't expose the port so we cant access this from local machine !

```
version: '3'
services: 
  postgres:
    image: 'postgres:latest'
    environment: 
      - POSTGRES_HOST_AUTH_METHOD=trust
  redis :
    image: 'redis:latest'
  server:
    build: 
      dockerfile: Dockerfile.dev
      context: ./server
    volumes: 
      - /app/node_modules
      - ./server:/app
    environment: 
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - PGUSER=postgres
      - PGHOST=postgres
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password
      - PGPORT=5432
    ports: 
      - "5000:5000"
```

