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
```sh
sudo apt-get update
sudp apt-install mysql-client-core-8.0
mysql -h 172.17.0.2 -u root -p
```
