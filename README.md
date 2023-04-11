<h1>Ansible. Практическое занятие</h1>
<h5>Задача: попрактиковаться в написании плейбука, со следующими условиями:
<li>подготовить стенд на Vagrant,</li>
<li>настроить ansible для доступа к стенду</li>
<li>написать плейбук, который устанавливает NGINX в конфигурации по умолчанию, с
  применением модуля yum</li>
<li>подготовить шаблон jinja2 новой конфигурации для nginx, чтобы сервис слушал на
нестандартном порту 8080. В шаблоне для номера порта использовать переменные
  ansible</li>
<li>добавить в плейбук копирование новой конфигурации сервиса на стенд</li>
<li>должен быть использован механизм notify для рестарта nginx после установки или
изменения конфигурации. </h5>
<h1>Установка Ansible</h1>
<h5>Если необходимо, установить ansible командой<h5>
<pre>sudo apt install ansible</pre>
Убедится в коррктночти установки:
  <pre>ansible --version</pre>
<h1>Подготовка стенда</h1>
В созданный каталог, скопировать присланные файлы, убедиться что файл Vagrant не содержит
ошибок. Проверяем статус Vagrant:
  <pre>vagrant status</pre>
Полнять Vagrant:
<pre>vagrant up</pre>
Для подключения к хосту nginx нам необходимо будет передать множество параметров - это
особенность Vagrant. Узнать эти параметры можно с помощью команды <pre>vagrant ssh-config</pre>.
<h1>Inventory</h1>
Создаем inventory файл с помощью команды <pre>nano inventory</pre>,вписываем в него параметры:
<pre>[webservers] nginx ansible_host=127.0.0.1 
ansible_port=2222 
ansible_private_key_file=/home/admin/os_lab2/test/ansible/.vagrant/machines/nginx/virtualbox/private_key</pre>
Вписываем свой порт, путь к приватному ключу ssh, эти данные мы получили после ввода команды <pre>vagrant ssh-config</pre>/
Убедимся, что Ansible может управлять нашим хостом. Сделать это можно с помощью команды:
<pre>ansible nginx -i inventory -m ping</pre>
ansible.cfg
  <h1>Playbook epel</h1>
Создаем файл epel.yml со следующим содержимым. Внимательно соблюдаем отступы, т. к. YaML очень чувствителен
к синтаксису:
<pre>---
- name: Install EPEL Repo
  hosts: webservers
  become: true
  tasks:
    - name: Install EPEL Repo package from standard repo
      yum:
        name: epel-release
        state: present
После чего запускаем выполнение Playbook:
ansible-playbook epel.yml </pre>
  <h1>Playbook nginx</h1>
За основу возьмем уже созданный плейбук epel.yml. Скопируем этот файл с именем nginx.yml. Добавим в этот новый файл установку пакета nginx. Секция будет выглядеть так:
<pre>---
- name: Install EPEL Repo
  hosts: webservers
  become: true
  vars:
    nginx_listen_port: 8080
  tasks:
    - name: Install EPEL Repo package from standard repo
      yum:
        name: epel-release
        state: present
    - name: install nginx from repo
      yum:
        name: nginx
        state: latest
      tags:
        nginx-package
        packages</pre>
Запустим только установку NGINX, использовав любой из подходящих тегов:
<pre>ansible-playbook nginx.yml --tag packages </pre>
<h1>Шаблон</h1>
Далее создадим файл шаблона для конфига NGINX, имя файла nginx.conf.j2.
<pre>events {
 worker_connections 1024;
}
http {
 server {
   listen {{ nginx_listen_port }} default_server;
   server_name default_server;
   root /usr/share/nginx/html;
   location / {
   }
 }
}</pre>
Добавим в playbook задачу, которая копирует подготовленный шаблон на хост. Также в плейбук добавлено определение переменной в секции vars. Для этого в nginx.yml введем следующее:
<pre>---
- name: Install EPEL Repo
  hosts: webservers
  become: true
  vars:
    nginx_listen_port: 8080
  tasks:
    - name: Install EPEL Repo package from standard repo
      yum:
        name: epel-release
        state: present
    - name: install nginx from repo
      yum:
        name: nginx
        state: latest
      tags:
        nginx-package
        packages
    - name: Create config file from template
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
     notify:
        - restart nginx
      tags:
        nginx-configuration</pre>
Запускаем playbook:
<pre>ansible-playbook nginx.yml </pre>
Добавить секции handler и notify для рестарта nginx не при любом старте playbook, а только при изменения в конфигурации. Для этого в nginx.yml введем следующее:
<pre>---
- name: Install EPEL Repo
  hosts: webservers
  become: true
  vars:
    nginx_listen_port: 8080
  tasks:
    - name: Install EPEL Repo package from standard repo
      yum:
        name: epel-release
        state: present
    - name: install nginx from repo
      yum:
        name: nginx
        state: latest
      tags:
        nginx-package
        packages
    - name: Create config file from template
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - restart nginx
      tags:
        nginx-configuration
  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes</pre>
Запускаем playbook:
<pre>ansible-playbook nginx.yml </pre>

Чтобы проверить работу NGINX на нестандартном порту, нам надо выяснить IP адрес, который получила виртуальная машина при создании vagrant-ом Это можно сделать командой:
<pre>vagrant ssh -c "ip addr show"</pre> 

В полученном выводе найдите описание сетевых интерфейсов. Первый интерфейс — локальный loopback, второй — автоматически добавленный вагрантом, предназначеный для интерконнекта ядра вагранта к виртуальной машине и не доступен снаружи виртуалки. А третий как раз публичный и к нему можно обращаться.
Затем в браузере открываем страницу:
http://192.168.11.150:8080/

