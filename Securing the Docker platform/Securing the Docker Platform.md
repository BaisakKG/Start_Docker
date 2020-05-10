# Securing the Docker Platform
В этом курсе будем рассматривать рекомендации по настройки хоста, Docker платформы и настройки Swarm
## Ресурсы для проверки рекомендаций
https://www.docker.com/legal/docker-cve-database
security@docker.com
forums.docker.com 
community.docker.com
CIS Docker Benchmark
https://www.inspec.io/
https://github.com/docker/docker-bench-security

## Проверка платформы Docker 
```
lsb_release -a
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security/
./docker-bench-security.sh -h
cat docker-bench-security.sh.log|less
```
## Оптимизация конфигурации платформы
```
head -18 functions_lib.sh
ls tests/ 
./docker-bench-security.sh -c check_1_2_8 #чтобы запустить определенный тест
./docker-bench-security.sh -c host_configuration #или выбрать раздел
```
* Список рекомендованных ОС для хоста Docker
	* CoreOS
	* Atomic
	* RancherOS
	* Ubuntu CoreOS
	* Photon OS
	* LinuxKit
	
## Установка RancherOS
1) Скачиваем с https://github.com/rancher/os/releases/ для VirtualBox rancheros-vmware.iso
2) Запускаемся с него (ОЗУ должен быть минимум 1300 мб) 
3) Создаем файл **cloud-config.yml**
4) Запускаем установку ``` sudo ros install -c cloud-config.yml -d /dev/sda ```
5) После установки выключаем ВМ, вынимаем rancheros-vmware.iso, включаем ВМ
```
 ps dockerd |grep proxy
 system-docker ps #все сервисы работают в контейнерах 
```
* User-Docker
user-docker используется для запуска контейнеров приложений и он настроен только для локального управления.
Настроим его для управления им через удаленный Докер клиент
```
#Создаем ключи для сервера и настриваем его
sudo ros tls gen -s -H $PUBLIC_IP $PUBLIC_HOSTNAME
sudo ros config set rancher.docker.tls true
sudo system-docker restart docker

#Создаем ключ для клиента
sudo ros tls gen

#Проверка 
docker --tlsverify version

#Теперь копируем ключи на докер клиент
scp rancher@$PUBLIC_IP:~/.docker/*.pem ~/.docker/

#Проверка подключения
export DOCKER_HOST=tcp://${PUBLIC_IP}:2376 DOCKER_TLS_VRIFY=1
docker info
docker container run -itd --rm -p 3000:80 -v /etc/hostname://etc/docker-hostname:ro --name nginxhello nbrown/nginxhello
```

## Audit Framework
* Kernel
	* auditctl
		audit.rules
	* auditd
		audit.conf
		* audit.log
			aureport
			ausearch

* Important Artifacts
	* Binaries
		- /usr/bin/dockerd (f)
		- /usr/bin/docker-containerd (f)
		- /usr/bin/docker-runc (f)
	* Config Files
		- /etc/default/docker (F)
		- /etc/docker/daemon.json (F)
	* System Unit Files 
		- docker.service (F)
		- docker.socket (F)
	* Execution Root
		- /var/lib/docker (D)
	* TLS Artifacts
		- /etc/docker (D)
```
pidof auditd
command -v auditd
sudo apt update
sudo apt install auditd
sudo aureport

#создаем первое правило которое будет следить за демоном докер, w(watch)-за кем смотреть и k(key)- как пометить
auditctl -w /usr/bin/dockerd -k docker 
auditctl -l 
dockerd -v 
aureport -k 
ausearch --event 422 #иднтификатор события
ausearch --event 422 | aureport -f -i 

files=("/var/lib/docker" "/etc/docker" "/lib/systemd/system/docker.service" "/lib/systemd/system/docker.socket" "/etc/default/docker" "/etc/docker/daemon.json" "/usr/bin/docker-containerd" "/usr/bin/docker-runc")
echo "${files[*]}"
for i in "${files[@]}"; do sudo auditctl -w $i -k docker; done
sudo auditctl -l

#сохраняем правила
sudo sh -c "auditctl -l >> /etc/audit/audit.rules"

./docker-bench-security.sh -c host_configuration
```

## Конфигурация Docker демона для оптимальной безопасности
```
./docker-bench-security.sh -c docker_daemon_configuration


