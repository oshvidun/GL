Для развернтываяния VM в GCP используем Ansible playbook
Сначала подготовим GCP, нам надо создать:
- Сервис аккаунт в GC с такими ролями Compute OS Admin Login, Editor, Service Account user
- Загрузить ключ .json
На хосте контролере:
- Установить Ansible
- ansible requests google-aut
- Google CLI
- Сгенерировать ключ ssh который будет добавлен в сервис акаунт 

плейбук будет иметь следующий вид
vars - описание переменных
для каждой ВМ создается отдельный ИП адрес котрый записывается в переменную, после чего мы создаем ВМ, добавляем диск, ИП адресс
post_tasks - добавляем задержку, чтобы инициализировалась служба ssh
````
# ansible notebook, to configure the VMs on GCP
- name: Create Compute Engine instances
  hosts: localhost
  gather_facts: no
  vars:
      gcp_project: task-5-372820
      gcp_cred_kind: serviceaccount
      gcp_cred_file: "/home/shvidun/ansible_automation/task-5-372820-0fb1361ce359.json"
      zone: "europe-west1-b"
      region: "europe-west1"
      machine_type: "n1-standard-1"
      image: "projects/rocky-linux-cloud/global/images/rocky-linux-8-v20220126"
      servers: "vm"
  tasks:
   - name: Create private IP address to the VM instance
     gcp_compute_address:
       name: "{{servers}}-ip"
       region: "{{ region }}"
       project: "{{ gcp_project }}"
       service_account_file: "{{ gcp_cred_file }}"
       auth_kind: "{{ gcp_cred_kind }}"
       wait: yes
     with_items: {{servers}}
     register: gce_ip
   - name: Bring up the instance in the zone
     gcp_compute_instance:
       name: "{{servers}}"
       machine_type: "{{ machine_type }}"
       disks:
         - auto_delete: true
           boot: true
           initialize_params:
             source_image: "{{ image }}"
       network_interfaces:
         - access_configs:
             - name: External NAT  # public IP
               nat_ip: "{{ gce_ip }}"
               type: ONE_TO_ONE_NAT
       zone: "{{ zone }}"
       project: "{{ gcp_project }}"
       service_account_file: "{{ gcp_cred_file }}"
       auth_kind: "{{ gcp_cred_kind }}"

     register: gce

  post_tasks:
    - name: Wait for SSH for instance
      wait_for: delay=5 sleep=5 host={{ gce_ip.address }} port=22 state=started timeout=100
    - name: Save host data for first zone
      add_host: hostname="vm1" groupname=gce_instances_ips
````

К сожелению мне пока не хватило времени разобратся как корректно создать цикл, чтобы создать сразу три инстанца (В идеии servers переменная должна содержать лист наименований ВМ, и цикл loop: {{servers|item}} ). 
Поэтому данный скрипт нам надо выполнить трижды

Создание  инвентори файла
Файл состоит из трех групп и одной метогруппы 
````
[vm1]
34.140.87.246
[vm2]
35.187.71.21
[vm3]
35.195.242.137
[iaas:children]
vm2
vm3
````
Чтобы проверить что все хорошо и наш контролер иммеет доступ к виртуальным машинам выполним команду
````
ansible all -i inventory -m ping
````

Создадим роли и плейбук для создания файла на ВМ группы iaas и сбора информации о всех инстансах
playbook
````
- name: create file
  hosts: iass
  gather_facts: yes
  roles:
    - create_file

- name: system info
  hosts: all
  gather_facts: true
  become: true
  roles:
    - system_info
````






