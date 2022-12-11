Для создания ВМ мы будем использовать az
1. Нам необходимо создать ресурс группу

	az group create --name GL-Task2 --location westeurope
	
	
2. Создание балансировщика нагрузки
	
	az network lb create \
		--resource-group GL-Task2 \
		--name GLTask2Balancer \
		--frontend-ip-name myFrontEndPool \
		--backend-pool-name myBackEndPool \
		--public-ip-address myPublicIP
		
3. Чтобы балансировщик нагрузки мог следить за состоянием приложения, необходимо настроить пробу работоспособности. Проба работоспособности динамически добавляет или удаляет виртуальные машины из балансировщика нагрузки на основе их ответа на проверки работоспособности. По умолчанию виртуальная машина удаляется из числа машин, на которые балансировщик распределяет нагрузку, после двух последовательных сбоев с интервалом в 15 секунд. Пробу работоспособности можно создать на основе протокола или конкретной страницы проверки работоспособности приложения.
	
	az network lb probe create \
		--resource-group GL-Task2 \
		--lb-name GLTask2Balancer \
		--name myHealthProbe \
		--protocol tcp \
		--port 80
		
4. Создадим правило балансировщика нагрузки
	
	az network lb rule create \
		--resource-group GL-Task2 \
		--lb-name GLTask2Balancer \
		--name myLoadBalancerRule \
		--protocol tcp \
		--frontend-port 80 \
		--backend-port 80 \
		--frontend-ip-name myFrontEndPool \
		--backend-pool-name myBackEndPool \
		--probe-name myHealthProbe
		
5. Нам необходимо сделать настройки виртуальной сети
	создание виртуальной сети
	az network vnet create \
		--resource-group GL-Task2 \
		--name myVnet \
		--subnet-name mySubnet
		
	создание группы безопасности сети
	az network nsg create \
		--resource-group GL-Task2 \
		--name myNetworkSecurityGroup
	
	создание правила группыбезоапсности
	az network nsg rule create \
		--resource-group GL-Task2 \
		--nsg-name myNetworkSecurityGroup \
		--name myNetworkSecurityGroupRule \
		--priority 1001 \
		--protocol tcp \
		--destination-port-range 80
		
	создаем 2 виртуальных сетевых аддаптера (по одной на каждую ВМ)	
	
	for i in `seq 1 2`; do
		az network nic create \
			--resource-group GL-Task2 \
			--name myNic$i \
			--vnet-name myVnet \
			--subnet mySubnet \
			--network-security-group myNetworkSecurityGroup \
			--lb-name GLTask2Balancer \
			--lb-address-pools myBackEndPool
	done
	
6. Создадим конфигурации ВМ используя cloud-init. Для этого в текущей оболочке создадим txt файл

#cloud-config
package_upgrade: true
packages:
  - nginx
  - nodejs
  - npm
write_files:
  - owner: www-data:www-data
  - path: /etc/nginx/sites-available/default
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection keep-alive;
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }
  - owner: azureuser:azureuser
  - path: /home/azureuser/myapp/index.js
    content: |
      var express = require('express')
      var app = express()
      var os = require('os');
      app.get('/', function (req, res) {
        res.send('Task 2 was performed by o.shvidun  ' + os.hostname() + '!')
      })
      app.listen(3000, function () {
        console.log('Hello world app listening on port 3000!')
      })
runcmd:
  - service nginx restart
  - cd "/home/azureuser/myapp"
  - npm init
  - npm install express -y
  - nodejs index.js
  
 7. Создадим группу доступности
 
	az vm availability-set create \
		--resource-group GL-Task2 \
		--name myAvailabilitySet
		
	Теперь мы можем создать виртуальные машины
	
	for i in `seq 1 2`; do
		az vm create \
			--resource-group GL-Task2 \
			--name myVM$i \
			--availability-set myAvailabilitySet \
			--nics myNic$i \
			--image UbuntuLTS \
			--admin-username azureuser \
			--generate-ssh-keys \
			--custom-data cloud-init.txt \
			--no-wait
	done
