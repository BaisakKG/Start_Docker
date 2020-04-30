```
docker version
docker info

docker images
docker pull alpine
docker pull ubuntu:16.04
docker rmi ubuntu:16.04 --- delete image

docker start cont
docker stop cont
docker rm cont
docker run -it --name temp ubuntu:latest /bin/bash
Ctrl P+Q --- выйти не убивая контейнер
docker stop $(docker ps -aq) --- чтобы остановить все контейнетры
docker rm $(docker ps -aq)
docker rmi $(docker images -q)
```
## Подготовка машин для Swarm
```
 lxc image copy images:alpine/3.11 local: --alias alp
 lxc launch alp mgr1
 lxc exec mgr1 -- ash
    apk update
    apk add docker
    rc-update add docker boot
lxc snapshot mgr1 mgr1-docker
lxc copy mgr1/mgr1-docker mgr2
lxc copy mgr1/mgr1-docker mgr3
lxc copy mgr1/mgr1-docker wrk1
lxc copy mgr1/mgr1-docker wrk2
lxc copy mgr1/mgr1-docker wrk3
lxc list
lxc start --all
lxc restart mgr1
lxc list
```
## Начинаем строить Swarm
```
lxc list -c n,4 --format csv > inventory

lxc exec mgr1 -- ash
    docker swarm init --advertise-addr 10.251.126.93:2377 --listen-addr 10.251.126.93:2377
    docker swarm join-token manager
    docker swarm join-token worker
    # через эти токены добавляются менеджеры и рабочие

lxc exec mgr2 -- ash
     docker swarm join --token SWMTKN-1-1cvmi32q8esq4ex56xw82yig5g4grj1ykuckzq34nsmymn3yss-4aup9hlgjrxbnv3wovusfh4wf 10.251.126.93:2377\
    --advertise-addr 10.251.126.184:2377\
    --listen-addr 10.251.126.184:2377

    docker node ls

lxc exec mgr3 -- ash
    docker swarm join --token SWMTKN-1-1cvmi32q8esq4ex56xw82yig5g4grj1ykuckzq34nsmymn3yss-8a3fbwhvcklt5sva898muz7th 10.251.126.93:2377\
    --advertise-addr 10.251.126.246:2377\
    --listen-addr 10.251.126.246:2377

lxc exec mgr1 -- ash
    # mgr3 у нас щас рабочий, сделаем его менеджером
    docker node ls
    docker node promote bfwqqz80sypys6ul0gn7s0wbl

lxc exec wrk1 -- docker swarm join --token SWMTKN-1-1cvmi32q8esq4ex56xw82yig5g4grj1ykuckzq34nsmymn3yss-8a3fbwhvcklt5sva898muz7th 10.251.126.93:2377\
    --advertise-addr 10.251.126.243:2377\
    --listen-addr 10.251.126.243:2377

lxc exec wrk2 -- docker swarm join --token SWMTKN-1-1cvmi32q8esq4ex56xw82yig5g4grj1ykuckzq34nsmymn3yss-8a3fbwhvcklt5sva898muz7th 10.251.126.93:2377\
    --advertise-addr 10.251.126.46:2377\
    --listen-addr 10.251.126.46:2377

lxc exec wrk3 -- docker swarm join --token SWMTKN-1-1cvmi32q8esq4ex56xw82yig5g4grj1ykuckzq34nsmymn3yss-8a3fbwhvcklt5sva898muz7th 10.251.126.93:2377\
    --advertise-addr 10.251.126.99:2377\
    --listen-addr 10.251.126.99:2377
```
## Service

```
docker service <create|ls|ps|inspect|update|rm>

mgr1
    docker service create --name psight1 -p 8080:8080 --replicas 5 nigelpoulton/pluralsight-docker-ci
    docker service ls
    docker service ps psight1
    docker service inspect psight1

```
### Scaling services

```
lxc exec mgr1 -- ash
    docker service ls
    docker service ps psight1
    docker service scale psight1=3 
    OR
    docker service update --replicas 3 psight1
```
### Rolling updates

```
lxc exec mgr1 -- ash
    docker service rm psight1
    docker service ls
    docker node ps self

    docker network create -d overlay ps-net
    docker network ls
    docker service create --name psight2 --network ps-net -p 80:80 --replicas 12 nigelpoulton/tu-demo:v1
    docker service ls
    docker service inspect --pretty psight2
```
**
docker service create --update-parallelism 2 --update-delay 10m
если указать эти параметры при создании сервиса при обновлении образа за раз будет обновлятся по два контейнера через каждые 10 минут
**

```
    docker service update --image nigelpoulton/tu-demo:v2 --update-parallelism 2 --update-delay 10s psight2
    docker service ps psight2 | grep v:2
    docker service inspect --pretty psight2
        Update status
        UpdateConfig
```

## Stack and DABs

У нас есть docker-compose.yml разными сервисами  и чтобы весь этот зоопарк запустить в Swam нужен Docker-compose чтобы преоброзовать yaml файл в DAB файл

```
lxc exec mgr1 -- ash
    docker-compose build
    docker images
    docker tag voteapp_result nigelpoulton/voteapp_result -- тегируем чтобы потом запушить в докер хаб
    docker login
    docker images
    docker push <image-name>
    #чтобы создать даб файл сперва отредактируем файл
    nano docker-compose.yml
        #здесь надо все строки buil: заменить на image: указать здесь на изображние которые мы запушили до этого
    
    docker-compose bundle
    docker stack deploy voteapp.dab --- stack и расширение можно не указывать
    docker service ls
    docker stack tasks voteapp
    docker service inspect voteapp_result
    docker stack rm voteapp
```
    

