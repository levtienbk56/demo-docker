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


