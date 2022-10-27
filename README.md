# demo-docker

## install Docker on EC2

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


## RUn Mysql Docker

create new volume for mysql on host EC2
```sh
docker volume create sqlvolume
docker volume ls
```
run a container that mount to above volume
```sh
docker run -d --mount volume=sqlvolume,target=/var/lib/mysql --name mysqlserver1 -e MYSQL_ROOT_PASSWORD=a123456 -e MYSQL_ROOT_HOST=% mysql
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


## Run Node api server

clone code from [here](https://gitlab.com/levtienbk56/nodejsmysql.git) into **$HOME/nodejsmysql** folder on host EC2.
use image  **node:14** to run a container that mount **$HOME/nodejsmysql** to **/home/app**, also map port `80:3000`
```
docker run -it -p 80:3000 -v $HOME/nodejsmysql:/home/app --name apiserver1 node:14 /bin/bash
```

edit parameters in file **server.js**:
```vim
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

access public IP of host EC2 [http://{public_ip_of_EC2}/users](http://{public_ip_of_EC2}/users)
```json
{"error":false,"data":[{"id":1,"name":"Max","email":"max@gmail.com","created_at":"2020-03-18T23:20:20.000Z"},{"id":2,"name":"John","email":"john@gmail.com","created_at":"2020-03-18T23:45:20.000Z"},{"id":3,"name":"David","email":"david@gmail.com","created_at":"2020-03-18T23:30:20.000Z"},{"id":4,"name":"James","email":"james@gmail.com","created_at":"2020-03-18T23:10:20.000Z"},{"id":5,"name":"Shaw","email":"shaw@gmail.com","created_at":"2020-03-18T23:15:20.000Z"},
```

