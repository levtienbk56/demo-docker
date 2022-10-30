# demo-docker

## [day1] Install Docker on EC2

prepare an Ubuntu EC2.
```sh
sudo apt-get update
sudo apt install debootstrap
```
get installation pack of Ubuntu 20.04 (name: focal) into focal folder.
```sh
sudo debootstrap focal focal
ls focal
```

install docker
```sh
sudo apt install docker.id -y
```

tar that focal folder then import file tar to create Docker Image
run a container from image
```sh
sudo tar -C focal -c . | sudo docker import - focal
```
check result with  
```sh
$ sudo docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
focal        latest    f1984c82a1e7   15 hours ago   322MB
```

now you can run a container from that image (-i: interactive, -t: output terminal, ls -a: the command which called when container finish boot)
```sh
sudo docker run -it focal ls -a
```

check status of list container on host EC2.
```sh
sudo docker container ls -a    //or sudo docker ps-a
```
Container will be stop/exited when its has no task. 
call container with `/bin/bash` to keep container running continuously.
use `CTR + D` to exit container.
```sh
sudo docker run -it focal /bin/bash
```

we can run docker without sudo by adding current user to docker group
```sh
sudo usermod -aG docker $USER
newgrp docker  //for changing group take effect
```


## [day2] Run Mysql Docker

create new volume for mysql on host EC2
```sh
docker volume create sqlvolume
docker volume ls
```
run a container that mount to above volume
```sh
docker run -d --mount source=sqlvolume,target=/var/lib/mysql --name mysqlserver1 -e MYSQL_ROOT_PASSWORD=a123456 -e MYSQL_ROOT_HOST=% mysql
```

connect to that container, then run mysql
```sh
docker exec -it mysqlserver1 /bin/bash
mysql -u root -p
show database;
ALTER USER 'root' IDENTIFIED WITH mysql_native_password BY '12356';
flush privileges;
exit;
```

on host EC2, let check container's IP, then connect to mysql. Try `show database;` to show all databases, then `exit;` (remember comma at the end of mysql command)
```
docker container inspect mysqlserver1
sudo apt-get update
sudp apt-install mysql-client-core-8.0
mysql -h 172.17.0.2 -u root -p
```


## [day3] Run Node api server

clone code from [here](https://gitlab.com/levtienbk56/nodejsmysql.git) into **$HOME/nodejsmysql** folder on host EC2.
use image  **node:14** to run a container that mount **$HOME/nodejsmysql** to **/home/app**, also map port `80:3000`
```
docker run -it -p 80:3000 -v $HOME/nodejsmysql:/home/app --name apiserver1 node:14 /bin/bash
```

edit parameters in file **server.js**:
```js
process.env.DB_HOST = '172.17.0.2';
process.env.DB_USER = 'root';
process.env.DB_PASS = '123456';
```
then run node server
```sh
npm install
node server.js
```
or define that parameter direct in run command
```
DB_HOST=172.17.0.2 DB_USER=root DB_PASS=123456 node server.js
```

check log of node server like this
```
Conneting to MYSQL IP= 172.17.0.2 USER=root PASS=123456
Node app is running on port 3000
MYSQL Connected!
 DB userapidb OK
 Select userapidb OK
DB Ready. App is running... 
```

access public IP of host EC2 [http://PUBLIC_IP_EC2/users](http://PUBLIC_IP_EC2/users)
```json
{"error":false,"data":[{"id":1,"name":"Max","email":"max@gmail.com","created_at":"2020-03-18T23:20:20.000Z"},{"id":2,"name":"John","email":"john@gmail.com","created_at":"2020-03-18T23:45:20.000Z"},{"id":3,"name":"David","email":"david@gmail.com","created_at":"2020-03-18T23:30:20.000Z"},{"id":4,"name":"James","email":"james@gmail.com","created_at":"2020-03-18T23:10:20.000Z"},{"id":5,"name":"Shaw","email":"shaw@gmail.com","created_at":"2020-03-18T23:15:20.000Z"},
```



## [day4] Dockerfile
Dockerfile is a script which contains a series of commands or instructions, to build the images.
the common of commands:
- **FROM**:  tells us what image to base this off of
- **COPY**:  It can copy a file (in the same directory as the Dockerfile) to the container
- **ARG**: This sets the environment variables, which can be used in the Dockerfile, only available during the build of a Docker image
- **ENV**:  This sets the environment variables, which can be used in the Dockerfile and any scripts that it calls. These are persistent with the container too, so they can be referenced at any time
- **VOLUME**: Specifying volume in the Dockerfile allows it to be externally mounted via the host itself or a Docker data container. If it’s not defined here, then it’s not possible to access outside of the container.
- **RUN**: this is what runs within the image at build time
- **CMD**: this is what runs within the container at build time
- **ENTRYPOINT**: same as CMD, which runs within container build time. if the **ENTRYPOINT** isn't specified, Docker will use /bin/sh -c as default. However, if you watnt to override some of the system defaults, you can specify your own entrypoint and therefore manipulate the environment. The ENTRYPOINT command also allows arguments to be set from the Docker run command as well, whereas CMD can’t.
- **USER**: create an user then login with this user when run container

[reference link](https://www.cloudbees.com/blog/what-is-a-dockerfile)

let use folder which has code from [day3] **$HOME/nodejsmysql**.
look over Dockerfile
```yml
FROM node:14

RUN apt-get update
WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .
EXPOSE 3000
CMD [ "node", "server.js" ]
```

let create docker image by command. (last . mean finding Dockerfile in current folder)
```
docker build -t mynodeapi .
docker images
```
then run a container (require run Mysqlserver at [day2] first)
```
docker run -d -p 3000:3000 mynodeapi /bin/bash
docker ps -a
```
container was be exited, cause of `/bin/bash` added to run command which override CMD `node server.js` in Dockerfile.
```
docker run -d -p 3000:3000 mynodeapi
docker ps -a
```
access to [http://EC2_PUBLIC_IP:3000/users](http://EC2_PUBLIC_IP:3000/users) to confirm nodeapi server working


## [day5] Docker Compose
Docker Compose provides a way to orchestrate multiple containers that work together(stack). Examples include a API service that processes requests and a Mysql database.
Docker Compose can create container from **Docker Images**, or from **Dockerfile**.
let do 3 examples:
1. Create container Mysql database from **docker image**
2. Create container Node API service from **Dockerfile**
3. Create both 2 containers Mysql and Node API at once

### prepare
create new 2 folders, for each service
```
UsersService
├── apiusers
└── mysql
```
install docker-compose tool
```sh
sudo apt  install docker-compose
```
and create new docker Network for stack which included both service
```
docker network create bridge500 --driver bridge --subnet=172.31.0.0/24
docker network inspect bridge500
```

### 1. Create container Mysql database from image

create docker volume to store database
```
docker volume create sqlvolume
```

move into folder `UsersService/mysql`, create new file **docker-compose.yml**
```yml
version: '3.5'
volumes:
  sqlvolume:
    external: true

services:
  mysql-server:
    image: mysql
    container_name: mysql-db-dev
    volumes:
      - sqlvolume:/var/lib/mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_DATABASE=userapidb
    networks:
      bridge500:
        ipv4_address: 172.31.0.2

networks:
  bridge500:
    external: true
```
run that docker-component.yml file to create container
```sh
docker-component up -d
docker ps -a
docker logs mysql-db-dev
```
verify IP address of container, then try connect to mysql database
```
docker container inspect mysql-db-dev
mysql -h 172.31.0.2 -u root userapidb -p
```

### 2. Create container Node API service from **Dockerfile**

move into folder `UsersService/apiusers` then clone code from [here](https://gitlab.com/levtienbk56/nodejsmysql.git)
```
git clone https://gitlab.com/levtienbk56/nodejsmysql.git .
```

prepare Dockerfile for Node API (create new if not exist)
```yml
FROM node:14

RUN apt-get update
WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .
EXPOSE 3000
CMD [ "node", "server.js" ]
```
then create new **docker-compose.yml** file:
```yml
version: '3.5'

services:
  webapi:
    build: .
    container_name: nodejs-api-dev
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=172.31.0.2
      - DB_USER=root
      - DB_PASS=123456
    networks:
      - bridge500

networks:
  bridge500:
    external: true
```
`build: . ` mean using Dockerfile in current folder.
`DB_HOST=172.31.0.2` is environment variable for using in container. (See `server.js` file).


run that docker-component.yml file to create container
```sh
docker-component up -d
docker ps -a
docker logs nodejs-api-dev
```
then access to [http://EC2_PUBLIC_IP:3000/users](http://EC2_PUBLIC_IP:3000/users) to confirm nodeapi server working.

let try edit `server.js`
```
return res.send({ error: true , message: 'Hello. This API service provides User CRUD.'})
```
after editing run below command to update change
```
docker-compose build
docker-compose up -d
```
then access to [http://EC2_PUBLIC_IP:3000/](http://EC2_PUBLIC_IP:3000/) to confirm change.


