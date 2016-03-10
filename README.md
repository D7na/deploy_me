## Кровью и потом: Деплой Rails приложения при помощи Ansible и Capistrano

В сети можно найти много инструкций по деплою rails-приложений на любой вкус. Но я все равно напишу еще_одну_инструкцию, потому-что не нашел в сети варианта, который бы сработал до конца для полного новичка. Летс го, май литл бразерс енд систерс!  

P.S. *в данном случае при установке и настройке используются следующие инструменты: rails 4, postgresql, rbenv, puma, nginx, capistrano. Если вы используете что-то другое, будте готовы к ошибкам.*  
P.P.S. *эта инструкция почти полностью копирует код из [этой](https://mkdev.me/posts/nastroyka-i-deploy-rails-prilozheniy-pri-pomoschi-ansible-i-capistrano) статьи, авторам которой я очень благодарен. Я заменил некоторые части и расписал неочевидные вещи.*

#### 0. Введение
Для деплоя используется новое пустое приложение с postgresql в качестве базы данных:
```bash
rails new deploy_me --database=postgresql
```
Сгенерируем что-нибудь через scaffold, чтобы хоть что-то видеть когда приложение запустится:
```bash
cd deploy_me
rails g scaffold User name:string email:string
```

#### 1. Подготовка сервера
На данном этапе предполагается что вы приобрели "сервер" и установили туда операционную систему. Обычно систему предлагают установить в процессе создания сервера. Например, я выбрал и установил Ubuntu 14.04. У вас на руках должны быть: IP-адрес сервера, имя пользователя (обычно root) и пароль.

###### 1.1 Настроим подключение к серверу с помощью ssh
Сгенерируем ключи (нам предложат выбрать место для хранения - лучше просто нажать enter, предложат добавить пароль к ключам - можно добавить):
```bash
ssh-keygen
```
Скопируем публичный ключ на наш сервер (вас спросят уверены ли вы - yes, попросят пароль пользователя (к серверу) - не отказывайте):
```bash
ssh-copy-id <имя_пользователя>@<ip-адрес>
```

Теперь мы можем заходить на наш сервер без ввода пароля:
```bash
ssh <имя_пользователя>@<ip-адрес>
```
Если не хотите каждый раз вводить имя пользователя и IP-адрес для этой команды, то можно создать удобное сокращение. Создайте (если его нет) и отредактируйте с помощью вашего любимого редактора файл **config**, который лежит в директории **~/.ssh**. Добавьте в него:
```text
Host <сокращение>
  Hostname <IP-адрес>
  User <имя_пользователя>
```
Теперь вы можете заходить на сервер так:
```bash
ssh <сокращение>
```

###### 1.2 Создадим с помощью ansible полноценную "инструкцию" (playbook) для настройки нашего сервера
Такой playbook хорош тем, что написав его один раз, вы сможете затем многократно использовать его для настройки других серверов. 

Установим Ansible:
```bash
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```

Проверим (ansible скажет свою версию):
```bash
ansible --version
```

Плейбук можно хранить где угодно, но я для удобства добавлю его в проект. В папке **config** я создам папку **server** а в ней наш плейбук **playbook.yml** и папку **configs** с конфигурациями, которые нужно добавить прежде чем плейбук будет запущен. Давайте взглянем на них:  
- database.yml
```yaml
production:
  adapter: postgresql
  username: <%= ENV["deploy_me_database_user"] %>
  password: <%= ENV["deploy_me_database_password"] %>
  pool: 8
  database: {{ dbname }}
```
Отсюда берутся настройки для подключения к базе данных. Здесь используются переменные окружения (также как, например, в secrets.yml), а также переменные которые использует ansible ( dbname ).
- .rbenv-vars
```text
SECRET_KEY_BASE=f3a7424bd0e4841635235a4e9a493d22a4ee26bf1bd8e26b94c9604d5c4b62a8f1aac725ae496159fb6bdbe2730d6ec9d130976527a390f04a5c59ca22723267
deploy_me_database_user={{ dbuser }}
deploy_me_database_password={{ dbpassword }}
```
Этот файл хранит все наши переменные окружения: dbuser и dbpassword автоматически вставит ansible, а вот SECRET_KEY_BASE нам обязательно нужно сгенерировать: запустите в корневой папке приложения
```bash
rake secret
```
и впишите полученный код вместо моего.
- settings.yml

Этот файл можно оставить пустым
- nginx.conf
```text
upstream backend {
  server unix:/home/{{ user }}/applications/{{ name }}/shared/tmp/sockets/puma.sock;
}

server {
  listen 80;

  root /home/{{ user }}/applications/{{ name }}/current/public;

  try_files $uri/index.html $uri.html $uri @{{ name }};

  location ~ ^/assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  location @{{ name }} {
    proxy_pass http://backend;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
  }
}
```
Это конфигурационный файл веб-сервера nginx. Переменные в нем также будут заменены ansible.

Я не стал ничего добавлять в .gitignore, чтобы все было видно в репозитории, но вы обязательно добавьте в него .rbenv-vars и playbook.yml или вообще всю папку service.

Playbook.yml с подробными комментариями. Его большая часть скопирована у авторов статьи - я вырезал проверки (теперь файл нужно применять только на чистом сервере), удалил копирование приватных ключей на сервер, изменил алгоритм настройки БД:
```yaml
---
- hosts: 'all'
  remote_user: 'root'

  # В данном блоке объявляются переменные, которые будут использоваться в playbook и конфигах, представленных выше
  vars:
    # Версия ruby
    ruby_version: '2.3.0'
    # Пользователь, от лица которого будет происходит деплой
    user: 'deploy'
    # Домашняя директория
    home: '/home/{{ user }}'
    # Директория установки Rbenv
    rbenv_root: '{{ home }}/.rbenv'
    # Название приложения
    name: 'deploy_app'
    # Путь до нашего приложения
    application: '{{ home }}/applications/{{ name }}'
    # Имя базы данных
    dbname: '{{ name }}_production'
    # Имя пользователя БД
    dbuser: '{{ user }}'
    # Пароль пользователя БД
    dbpassword: '12345678'

  # Список задач, которые будут выполнены последовательно
  tasks:
    # Обновление кеша
    - name: 'apt | update'
      action: 'apt update_cache=yes'

    # Установка пакетов, необходимых для сервера
    - name: 'apt | install dependencies'
      action: 'apt pkg={{ item }}'
      with_items:
        - 'build-essential'
        - 'libssl-dev'
        - 'libyaml-dev'
        - 'libreadline6-dev'
        - 'zlib1g-dev'
        - 'libcurl4-openssl-dev'
        - 'git'
        - 'nginx'
        - 'redis-server'
        - 'postgresql'
        - 'postgresql-contrib'
        - 'libpq-dev'
        - 'imagemagick'
        - 'libmagickwand-dev'
        - 'nodejs'
        - 'htop'
        - 'autoconf'
        - 'bison'
        - 'libncurses5-dev'
        - 'libffi-dev'
        - 'libgdbm3'
        - 'libgdbm-dev'
        - 'python-psycopg2'

    # Сгенерируем локали чтобы сервер не ругался
    - name: 'fix locales'
      shell: 'locale-gen en_US.UTF-8 && locale-gen ru_RU.UTF-8 && dpkg-reconfigure locales'

    # Создаём нашего пользователя deploy
    - name: 'account | create'
      user: 'name={{ user }} shell=/bin/bash'

    # Копируем авторизованный ключ
    - name: 'account | copy authorized keys'
      shell: 'mkdir -p {{ home }}/.ssh -m 700 && cp /root/.ssh/authorized_keys {{ home }}/.ssh && chown -R {{ user }}:{{ user }} {{ home }}/.ssh'

    # Устанавливаем rbenv
    - name: 'rbenv | clone repo'
      git: 'repo=git://github.com/sstephenson/rbenv.git dest={{ rbenv_root }} accept_hostkey=yes'

    - name: 'rbenv | add bin to path'
      shell: echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> {{ home }}/.bashrc

    - name: 'rbenv | init'
      shell: echo 'eval "$(rbenv init -)"' >> {{ home }}/.bashrc

    # Устанавливаем плагины rbenv - ruby-build и rbenv-vars
    - name: 'rbenv | clone ruby-build repo'
      git: 'repo=git://github.com/sstephenson/ruby-build.git dest={{ rbenv_root }}/plugins/ruby-build accept_hostkey=yes'

    - name: 'rbenv | clone rbenv-vars repo'
      git: 'repo=https://github.com/sstephenson/rbenv-vars.git dest={{ rbenv_root }}/plugins/rbenv-vars accept_hostkey=yes'

    # Создаем shared-директорию для файлов конфигурации
    - name: 'all | create shared directory'
      shell: 'mkdir -p {{ application }}/shared/config -m 775 && chown -R {{ user }}:{{ user }} {{ home }}/applications'

    # Копируем .rbenv-vars с нашими переменными окружения
    - name: 'rails | copy .rbenv-vars'
      template: 'src=configs/.rbenv-vars dest={{ application }}/shared/.rbenv-vars owner={{ user }} group={{ user }} mode=0644'

    # Устанавливаем Ruby
    - name: 'rbenv | install ruby {{ ruby_version }}'
      shell: 'RBENV_ROOT={{ rbenv_root }} PATH="$RBENV_ROOT/bin:$PATH" rbenv install {{ ruby_version }}'

    # Настраиваем rbenv, устанавливаем bundler
    - name: 'rbenv | set global ruby {{ ruby_version }}'
      shell: 'RBENV_ROOT={{ rbenv_root }} PATH="$RBENV_ROOT/bin:$PATH" rbenv global {{ ruby_version }}'

    - name: 'rbenv | rehash'
      shell: 'RBENV_ROOT={{ rbenv_root }} PATH="$RBENV_ROOT/bin:$PATH" rbenv rehash'

    - name: 'rbenv | create .gemrc'
      lineinfile: 'dest={{ home }}/.gemrc owner={{ user }} group={{ user }} mode=0644 line="gem: --no-ri --no-rdoc" create=yes'

    - name: 'ruby | install bundler'
      shell: 'RBENV_ROOT={{ rbenv_root }} PATH="$RBENV_ROOT/bin:$PATH" rbenv exec gem install bundler'

    - name: 'rbenv | change owner'
      shell: 'chown -R {{ user }}:{{ user }} {{ rbenv_root }}'

    # Настраиваем postgresql - создаем базу данных, добавляем пользователя и его привилегии
    - name: 'create database'
      become: true
      become_user: postgres
      postgresql_db: name={{dbname}}

    - name: 'create db user and grant access to database'
      become: true
      become_user: postgres
      postgresql_user: db={{dbname}} name={{dbuser}} password={{dbpassword}} priv=ALL

    - name: 'ensure user does not have unnecessary privilege'
      become: true
      become_user: postgres
      postgresql_user: name={{dbuser}} role_attr_flags=NOSUPERUSER,NOCREATEDB

    - name: 'postgresql | restart service'
      service: name=postgresql state=restarted

    # Копируем database.yml
    - name: 'postgresql | copy database.yml'
      template: 'src=configs/database.yml dest={{ application }}/shared/config/database.yml owner={{ user }} group={{ user }} mode=0644'

    # Копируем settings.yml
    - name: 'rails | copy settings.yml'
      copy: 'src=configs/settings.yml dest={{ application }}/shared/config/settings.yml owner={{ user }} group={{ user }} mode=0644'

    # Настраиваем Nginx: удаляем дефолтный файл, копируем свой конфиг, перезапускаем
    - name: 'nginx | createdir'
      shell: 'rm /etc/nginx/sites-enabled/default; mkdir -p etc/nginx/sites-enabled/'

    - name: 'nginx | copy config'
      template: 'src=configs/nginx.conf dest=/etc/nginx/sites-enabled/{{ name }}.conf owner=root group=root mode=0644'

    - name: 'nginx | restart service'
      service: name=nginx state=restarted

```
Теперь перейдем в папку config/server и запустим наш плейбук 
```bash
ansible-playbook -i<IP-адрес>, playbook.yml
```
Если все прошло без ошибок, значит сервер готов принять rails и мы даже можем ввести в браузере наш IP-адрес и увидеть что Nginx работает, весело показывая нам свой Bad Gateway.

P.S. *На слабых машинах установка может занимать длительное время или вообще не дойти до конца. К примеру одноядерный VDS с 512мб оперативной памяти никак не мог поставить Ruby. Переход на тариф "2 ядра, 2Гб ОЗУ" решил проблему.*   

#### 2. Подготовка приложения и деплой
Добавим в гемфайл следующие строки:
```gemfile
gem 'puma'
group :development do
  # Гем, который добавляет специфические для Rails таски, такие как прогон миграций и компиляция ассетов
  gem 'capistrano-rails'
  # Гем, добавляющий возможности bundle к capistrano
  gem 'capistrano-bundler'
  # Добавление поддержки Rbenv (менеджера версий для Ruby)
  gem 'capistrano-rbenv'
  # Интеграция пумы и капистрано
  gem 'capistrano3-puma'
end
```
Перейдем в корневую папку приложения и запустим, сначала bundler
```bash
bundle install
```
затем установку capistrano
```bash
cap install
```
Capistrano сгенерирует файлы, которые нам необходимо отредактировать:

- config/deploy/production.rb:
```ruby
ip = '<IP-адрес>'

role :app, ["deploy@#{ip}"]
role :web, ["deploy@#{ip}"]
role :db,  ["deploy@#{ip}"]

server ip, user: 'deploy', roles: %w{web app db}

set :stage, 'production'
set :rails_env, 'production'
```
- config/deploy.rb
```ruby
# config valid only for current version of Capistrano
lock '3.4.0'

set :application, 'deploy_me'
set :repo_url, 'https://github.com/robot-den/deploy_me.git'

# Default branch is :master
# ask :branch, `git rev-parse --abbrev-ref HEAD`.chomp

# Default deploy_to directory is /var/www/my_app_name
set :deploy_to, '/home/deploy/applications/deploy_me'

# Default value for :scm is :git
# set :scm, :git

# Default value for :format is :pretty
# set :format, :pretty

# Default value for :log_level is :debug
set :log_level, :info

# Default value for :pty is false
# set :pty, true

# Копирующиеся файлы и директории (между деплоями)
set :linked_files, %w{config/database.yml config/settings.yml .rbenv-vars}
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/uploads}

# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }

# Default value for keep_releases is 5
# set :keep_releases, 5

# Ruby свистелки
set :rbenv_type, :user
set :rbenv_ruby, '2.3.0'
set :rbenv_prefix, "RBENV_ROOT=#{fetch(:rbenv_path)} RBENV_VERSION=#{fetch(:rbenv_ruby)} #{fetch(:rbenv_path)}/bin/rbenv exec"
set :rbenv_roles, :all

# А это рекомендуют добавить для приложений, использующих ActiveRecord
set :puma_init_active_record, true

namespace :deploy do

  after :restart, :clear_cache do
    on roles(:web), in: :groups, limit: 3, wait: 10 do
      # Here we can do anything such as:
      # within release_path do
      #   execute :rake, 'cache:clear'
      # end
    end
  end

end
```
- Capfile
```ruby
# Load DSL and set up stages
require 'capistrano/setup'

# Include default deployment tasks
require 'capistrano/deploy'

require 'capistrano/rbenv'
require 'capistrano/bundler'
require 'capistrano/rails'
require 'capistrano/puma'

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

Также нам необходимо создать файл конфигурации для puma - puma.rb в папке config:
```ruby
# Change to match your CPU core count
workers 2

# Min and Max threads per worker
threads 1, 6

app_dir = File.expand_path("../..", __FILE__)
shared_dir = "/home/deploy/applications/deploy_app/shared"

# Default to production
rails_env = ENV['RAILS_ENV'] || "production"
environment rails_env

# Set up socket location
bind "unix://#{shared_dir}/tmp/sockets/puma.sock"

# Logging
stdout_redirect "#{shared_dir}/log/puma.stdout.log", "#{shared_dir}/log/puma.stderr.log", true

# Set master PID and state locations
pidfile "#{shared_dir}/tmp/pids/puma.pid"
state_path "#{shared_dir}/tmp/pids/puma.state"
activate_control_app

on_worker_boot do
  require "active_record"
  ActiveRecord::Base.connection.disconnect! rescue ActiveRecord::ConnectionNotEstablished
  ActiveRecord::Base.establish_connection(YAML.load_file("#{app_dir}/config/database.yml")[rails_env])
end
```
Вам нужно отредактировать эти файлы в соответствии с вашими данными (например IP-адрес, адрес репозитория). Обратите внимание, что если вы используете имя приложения, имя пользователя или названия директорий отличные от моих, то вам необходимо соответственно изменить эти конфигурационные файлы. Шоу "интуиция" начинается !

Теперь можно закоммитить изменения в наш репозиторий, который мы уже указывали в deploy.rb:   

Для справки, в корневой папке приложения:
```bash
git init
git add .
git commit -m "initial commit"
git remote add origin <адрес вашего гит-репозитория>
git push origin master
```

Произведем проверку что всего хватает: 
```bash
cap production deploy:check
```
Запустим деплой 
```bash
cap production deploy
```
Разбудим пуму:
```bash
cap production puma:start
```
Ура! Теперь можно перейти на наш IP адрес и увидеть ошибку от rails! Можно дописать к адресу /users и увидеть что-то рабочее.
