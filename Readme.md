## Cоздать сервис, состоящий из 2 различных контейнеров: 1 - веб, 2 - БД
Создаём файл **docker-compose.yaml**, который будет использоваться для запуска двух связанных сервисов - phpmyadmin и mysql. Эти сервисы будут работать вместе, а для управления БД будет использоваться веб-интерфейс phpMyAdmin
```
vi docker-compose.yml
```
Содержимое файла:
```
version: '3'

services:
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    ports:
      - "8080:80"
    environment:
      PMA_HOST: mysql

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: 12345
    volumes:
      - ./data:/var/lib/mysql
```
Сервис **phpmyadmin** использует образ phpmyadmin/phpmyadmin и пробрасывает порт 80 контейнера на порт 8080 хоста. Также задается переменная окружения PMA_HOST со значением mysql, которая указывает на имя сервиса mysql внутри сети Docker.

Сервис **mysql** использует образ mysql:5.7 и задает переменную окружения MYSQL_ROOT_PASSWORD со значением 12345. Также задается том ./data:/var/lib/mysql, который монтируется внутрь контейнера и используется для хранения данных базы данных.

Осталось запустить сервисы с помощью команды
```
docker-compose up -d
```
```
docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED             STATUS          PORTS                                   NAMES
39a50e49f2da   mysql:5.7               "docker-entrypoint.s…"   About an hour ago   Up 6 seconds    3306/tcp, 33060/tcp                     container_hw5_mysql_1
4fbbd943250a   phpmyadmin/phpmyadmin   "/docker-entrypoint.…"   About an hour ago   Up 6 seconds    0.0.0.0:8080->80/tcp, :::8080->80/tcp   container_hw5_phpmyadmin_1
```
Рисунок phpmyadmin.png

## Cоздать 3 сервиса в каждом окружении (dev, prod, lab). По итогу на каждой ноде должно быть по 2 работающих контейнера

**Задание выполняется из рассчета, что у меня две виртуалки node1 (192.168.1.7) и node2 (192.168.1.8)**

На первой (node1) выполняем команду, которая инициализирует режим управления кластером Docker Swarm на текущем узле.:
```
[ubuntu@node1 ~]$  docker swarm init
Swarm initialized: current node (sujt19duqiszzlgxnyev8cb1x) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3iccdphpx626mlcyg1vdwfpap0mwch8aucfzkrexckvocgip55-3ny2t9wnqoggr8dxfcukrtoiw 192.168.1.7:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
На второй (node2) виртуалке выполняем команду, которая присоединяет узел node2 к кластеру Docker Swarm с помощью токена, который был сгенерирован при инициализации кластера на узле node1:
```
[ubuntu@node2 ~]$  docker swarm join --token SWMTKN-1-3iccdphpx626mlcyg1vdwfpap0mwch8aucfzkrexckvocgip55-3ny2t9wnqoggr8dxfcukrtoiw 192.168.1.7:2377
This node joined a swarm as a worker.
```
Выведем список всех узлов в кластере Docker Swarm:
```
[ubuntu@node1 ~]$  docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
sujt19duqiszzlgxnyev8cb1x *   node1      Ready     Active         Leader           24.0.4
qqje7q696npjb1i2s5emz4xf1     node2      Ready     Active                          24.0.4
```

Создаём сеть типа overlay для связывания сервисов, контейнеры которых находятся на разных нодах:
```
docker network create --driver overlay test-network --attachable
```
Создаём сервис **mariadb_service** c одним контейнером
```
[ubuntu@node1 ~]$ docker service create --name mariadb_service --replicas 1 -e MYSQL_ROOT_PASSWORD=12345 --network test-network mariadb:10.10.2
oi48ghs29dqeyzds64tbiylq7
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```
И создаём сервис **adminer_service**
```
[ubuntu@node1 ~]$ docker service create --name adminer_service --replicas 1 -p 6081:8080 --network test-network adminer:4.8.1
g4s0kg1619km1yk706u4djja3
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```
Доступ работает с обеих нод:

Рисунок node1_admin.png

Рисунок node2_admin.png

Навешиваем ярлык "dev" на node2:
```
docker node update --label-add env=dev node2
```
Теперь в кластере Docker Swarm выполняем команду:
```
[ubuntu@node1 ~]$  docker service create --name mariadb_service_dev_node2 --constraint node.labels.env==dev --replicas 2 -e MYSQL_ROOT_PASSWORD=12345 -p 3306:3306 mariadb:10.10.2
p5jdou264t6vkr2zcwdo20pdw
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged
```
Эта команда создаёт новый сервис с именем mariadb_service_dev_node2, запускает две реплики контейнера, которые будут размещены только на узлах, у которых есть метка env со значением dev (это у нас node2). Также команда задает переменную окружения MYSQL_ROOT_PASSWORD со значением 12345, пробрасывает порт 3306 контейнера на порт 3306 хоста и использует образ mariadb:10.10.2.

Выводим список всех сервисов, запущенных в кластере Docker Swarm
```
[ubuntu@node1 ~]$  docker service ls
ID             NAME                        MODE         REPLICAS   IMAGE             PORTS
g4s0kg1619km   adminer_service             replicated   1/1        adminer:4.8.1     *:6081->8080/tcp
oi48ghs29dqe   mariadb_service             replicated   1/1        mariadb:10.10.2
p5jdou264t6v   mariadb_service_dev_node2   replicated   2/2        mariadb:10.10.2   *:3306->3306/tcp
```
На node2:
```
[ubuntu@node2 ~]$  docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED              STATUS              PORTS      NAMES
b5ac9d3eaf94   mariadb:10.10.2   "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp   mariadb_service_dev_node2.1.zrihjljipks98nt5kf2cs1szh
20839cf479ef   mariadb:10.10.2   "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp   mariadb_service_dev_node2.2.myb624y0h8k1vfcfffsby91nj
3dd5b2e5183f   adminer:4.8.1     "entrypoint.sh php -…"   2 minutes ago        Up 2 minutes        8080/tcp   adminer_service.1.ninp9m5a9xlt5j9uoi6rzn7c2
```

Эскалируем и меняем количество реплик сервиса mariadb_service до 3:
```
[ubuntu@node1 ~]$  docker service scale mariadb_service_dev_node2=3
mariadb_service_dev_node2 scaled to 3
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged

[ubuntu@node1 ~]$  docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED         STATUS         PORTS      NAMES
c3c5b3c9be39   mariadb:10.10.2   "docker-entrypoint.s…"   7 minutes ago   Up 7 minutes   3306/tcp   mariadb_service.1.0v5yk4r90gapsj1erbyf4k37q
```
На node2:
```
[ubuntu@node2 ~]$  docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED              STATUS              PORTS      NAMES
d0fceeadbb5a   mariadb:10.10.2   "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp   mariadb_service_dev_node2.3.jorxjcyfp4ke8y26vbip36023
b5ac9d3eaf94   mariadb:10.10.2   "docker-entrypoint.s…"   5 minutes ago        Up 5 minutes        3306/tcp   mariadb_service_dev_node2.1.zrihjljipks98nt5kf2cs1szh
20839cf479ef   mariadb:10.10.2   "docker-entrypoint.s…"   5 minutes ago        Up 5 minutes        3306/tcp   mariadb_service_dev_node2.2.myb624y0h8k1vfcfffsby91nj
3dd5b2e5183f   adminer:4.8.1     "entrypoint.sh php -…"   6 minutes ago        Up 6 minutes        8080/tcp   adminer_service.1.ninp9m5a9xlt5j9uoi6rzn7c2
```
Выведем список всех заданий, связанных с сервисом mariadb_service и их текущий статус:
```
[ubuntu@node1 ~]$  docker service ps mariadb_service
ID             NAME                IMAGE             NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
0v5yk4r90gap   mariadb_service.1   mariadb:10.10.2   node1     Running         Running 8 minutes ago
```
Аналогично для сервиса mariadb_service_dev_node2:
```
[ubuntu@node1 ~]$  docker service ps mariadb_service_dev_node2
ID             NAME                          IMAGE             NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
zrihjljipks9   mariadb_service_dev_node2.1   mariadb:10.10.2   node2     Running         Running 7 minutes ago
myb624y0h8k1   mariadb_service_dev_node2.2   mariadb:10.10.2   node2     Running         Running 7 minutes ago
jorxjcyfp4ke   mariadb_service_dev_node2.3   mariadb:10.10.2   node2     Running         Running 3 minutes ago
```