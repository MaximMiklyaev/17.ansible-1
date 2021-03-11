#ansible-
          
#версия Ansible  =>2.4 требует для своей работы Python 2.6 или выше
          
#убедитесь что у Вас установлена нужная версия

          python -V

#настройка Ansible

#для управления хостами Ansible использует SSH соединение. Поэтому перед стартом необходимо убедиться что у Вас есть доступ до управляемых хостов.

#также на управляемых хостах должен быть установлен Python 2.X

#подготовка окружения

#создайте каталог ansible и положите в него Vagrantfile

#поднимите управляемый хост командой 

         vagrant up

#подключитесь к VM

          vagrant ssh

#для подключения к хосту nginx нам необходимо будет передать множество параметров - это особенность Vagrant. Узнать эти параметры можно с помощью команды
 
          vagrant ssh-config

#пример основных необходимых нам

          vagrant ssh-config

#вывод

          Host ansible
            HostName 127.0.0.1
            User vagrant
            Port 2200
            UserKnownHostsFile /dev/null
            StrictHostKeyChecking no
            PasswordAuthentication no
            IdentityFile /17.ansible-1/.vagrant/machines/ansible/virtualbox/private_key
            IdentitiesOnly yes
            LogLevel FATAL

#используя эти параметры создадим свой первый inventory файл. выглядеть он будет так

          [web]nginx ansible_host=127.0.0.1 ansible_port=2200 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/ansible/virtualbox/private_key

#написания Playbook-а для установки NGINX.

#за основу возьмем уже созданный нами файл epel.yml (я его переименую в nginx.yml). и первым делом добавим в него установку пакета NGINX. 

#секция будет выглядеть так

          ame: Install nginx package from epel repo yum: name: nginx state: latest tags: - nginx-package Как видите добавлены tags - packages 

#Обратите внимание - добавили Tags . теперь можно вывести в консоль список тегов и выполнить, например, только установку NGINX. в нашем случае так, например, можно осуществлять его обновление.

#выведем в консоль все теги

          ansible-playbook nginx.yml --list-tags playbook: epel.yml play #1 (nginx): NGINX | Install and configure NGINX TAGS: [] TASK TAGS: [epel-package, nginx-package, packages]

#запустим только установку NGINX:

          ansible-playbook nginx.yml -t nginx-package

#далее добавим шаблон для конфига NGINX и модуль, который будет копировать этот шаблон на хост

          name: NGINX | Create NGINX config file from template template: src: templates/nginx.conf.j2 dest: /tmp/nginx.conf tags: - nginx-configuration 

#сразу же пропишем в Playbook необходимую нам переменную. Нам нужно чтобы NGINX слушал на порту 8080

          name: NGINX | Install and configure NGINX hosts: nginx become: true vars: nginx_listen_port: 8080

#в итоге на данном этапе Playbook будет выглядеть так Добавлена только секция vars

#сам шаблон будет выглядеть так

          events {
             worker_connections 1024;
          }
          http {
             server {
                 listen       {{ nginx_listen_port }} default_server;
                 server_name  default_server;
                 root         /usr/share/nginx/html;
                 location / {
                 }
             }
          }


#теперь создадим handler и добавим notify к копированию шаблона. Теперь каждый раз когда конфиг будет изменяться - сервис перезагрузиться. Секция с handlers будет выглядеть следующим образом

           handlers:
              name: restart nginx
               systemd:
                 name: nginx
                 state: restarted
                 enabled: yes
              name: reload nginx
               systemd:
                 name: nginx
                 state: reloaded

#так же создадим handler для рестарта и включения сервиса при загрузке Перечитываем конфиг 

#notify будут выглядеть так
 
          name: NGINX | Install NGINX package from EPEL Repo
               yum:
                 name: nginx
                 state: latest
               notify:
          restart nginx
               tags:
          nginx-package
          packages
          name: NGINX | Create NGINX config file from template
               template:
                 src: templates/nginx.conf.j2
                 dest: /etc/nginx/nginx.conf
               notify:
          reload nginx
               tags:
          nginx-configuration

#результирующий файл nginx.yml, теперь можно его запустить

          ansible-playbook playbooks/nginx.yml

#переходим в браузере по адресу 

          http://192.168.11.101:8080

#убеждаемся что сайт доступен

#из консоли выполнить команду

          curl http://192.168.11.101:8080
