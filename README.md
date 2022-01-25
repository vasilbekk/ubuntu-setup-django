# Инструкция по настройке Ubuntu-сервера для Django
Данное руководство поможет вам настроить Ubuntu-server для запуска Django проектов в боевом режиме! Приятного прочтения.


## Обновляем пакеты и пакетный менеджер. Устанавливаем полезные утилиты.
```
sudo apt-get update
```
```
sudo apt-get install -y vim htop tree git curl zsh tmux nginx python3.10 python3-pip python3-venv
```


Утилита  | Назначение
------------- | -------------
`vim`  | Текстовый редактор
`htop`  | Просмотр запущенных процессов, информации о них
`tree`  | Вывод каталогов в удобном формате
`git`  | Работа с [GitHub](https://github.com/)
`curl`  | Отправка Http-запросов
`zsh`  | Супер командная оболочка UNIX, делает работу с консолью удобнее
`tmux`  | Мультиплексор, несколько терминалов в одном окне, также есть встроенная альтернатива screen
`nginx`  | Веб-сервер
`python3.8`  | Python версии 3.8.x
`python3-pip`  | Менеджер пакетов для Python
`python3-venv`  | Виртуальное окружение Python

## Добавляем нового пользователя. Настраиваем доступ к серверу.
1. Создаём пользователя `www` --> Придумываем пароль --> Данные заполнять необязательно (Full Name, Room Number, ..., other).
2. Добавляем пользователя `www` в группу `sudo`, чтобы получить права администратора.
3. Отсоединяемся от сервера.
4. Копируем SSH-ключ на сервер для нового пользоветеля.
5. Подключаемся к серверу под новым пользователем.

```
sudo adduser www
sudo adduser www sudo
exit
ssh-copy-id www@<адрес сервера>
ssh www@<адрес сервера>
```
Открываем SSH-конфиг и изменяем следующие параметры:
1. Разрешаем подключаться по SSH только пользователю `www`
2. Запрещаем подключаться пользователю `root`
3. Запрещаем подключаться по паролю
```
sudo vim /etc/ssh/sshd_config
    AllowUsers www
    PermitRootLogin no
    PasswordAuthentication no
```
Перезапускаем SSH-клиент
```
sudo service ssh restart
```

## Настраиваем ZSH и устанавливаем удобную оболочку
Скачиваем и устанавливаем [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh) (Конфигурация ZSH). Делаем его оболочкой по умолчанию.
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sudo chsh -s $(which zsh)
```
Добавляем шорткаты для удобный работы с терминалом
```
vim ~/.zshrc
    alias cls='clear'
    alias start_django='python3 manage.py runserver 0.0.0.0:8000'
    alias sd='start_django'
    alias activate_venv='source venv/bin/activate'
```
Перезапускаем ZSH-клиент
```
. ~/.zshrc
```
## Клонируем Django-проект и настраиваем виртуальное окружение
Клонируем репозиторий с проектом
```
mkdir project && cd project
git clone https://github.com/<username>/<repository_name>
```
Устанавливаем `virtualenv`, создаем виртуальное окружение и активируем его
```
pip3 install virtualenv
python3 -m venv venv
source venv/bin/activate
```
Устанавливаем необходимые пакеты (в моем случае все пакеты хранятся в requirements.txt)
```
pip3 install -r requirements.txt
```
## Настройка Gunicorn
Устанавливаем `gunicorn` и сохраняем список установленных пакетов
```
pip3 install gunicorn
pip3 freeze > requirements.txt
```
Создаем Gunicorn-конфиг
```
vim /home/www/project/<repository_name>/gunicorn_config.py
```
Заполняем параметры Gunicorn-конфига
```
command = '/home/www/project/venv/bin/gunicorn'
pythonpath = '/home/www/project/<repository_name>'
bind = '127.0.0.1:8001'
workers = <2*(Кол-во_ядер)-1>
user = 'www'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=<repository_name>.settings'
```
Создаём bash-скрипт для запуска Gunicorn'a
```
mkdir bin
vim bin/start_gunicorn.sh
	#!/bin/bash
	source /home/www/project/venv/bin/activate
	exec gunicorn  -c "/home/www/project/<repository_name>/gunicorn_config.py" <repository_name>.wsgi

```
Проверяем работу Gunicorn'a, если нет ошибок, значит вы всё сделали правильно
```
bin/start_gunicorn.sh
```
Вот мое древо директорий папки `project`
```
├── bin
│   └── start_gunicorn.sh
├── requirements.txt
├── venv  
└── repository_name
    ├── gunicorn_config.py
    ├── manage.py
    ├── repository_name
    └── ...
```
## Настройка Nginx
1. Создаем директорию log-файлов
2. Удаляем и создаем чистый Nginx-конфиг
```
mkdir logs
sudo rm /etc/nginx/sites-enabled/default
sudo vim /etc/nginx/sites-enabled/default
```
Вставляем данный шаблон
```
server {
        listen 80 default_server;
        listen [::]:80 default_server;
	error_log /home/www/project/logs/nginx_errors.log;
	access_log /home/www/project/logs/nginx_access.log;
        root /var/www/html;
	
	client_max_body_size 100M;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
        	proxy_pass http://127.0.0.1:8001;
        	proxy_set_header X-Forwarded-Host $server_name;
        	proxy_set_header X-Real-IP $remote_addr;
        	add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
        	add_header Access-Control-Allow-Origin *;
        }

}
```
Перезапускаем Nginx-клиент
```
sudo service nginx restart
```
Если вы введете в браузере адрес сервера, то должны получить ошибку: "502 Bad Gateway", так как Gunicorn пока что отключён и Nginx'у некуда проксировать запросы.

## Запуск
Для запуска активируем созданный раннее bash-скрипт
```
bin/start_gunicorn.sh
```
Теперь при переходе на сайт всё должно работать, но без `staticfiles`. Их нужно подключать отдельно.
Не забудьте изменить settings.py
```
vim <repository_name>/settings.py
	...
	DEBUG=False
	...
```

## Подключение Django StaticFiles и MediaFiles
Переходим в репозиторий и собираем `staticfiles` в одну директорию
```
python3 manage.py collectstatic
```
Теперь все статичные файлы хранятся в нашем репозитории в папке `static`. Открываем Nginx-конфиг и добавляем параметры проксирования запросов.
```
sudo vim /etc/nginx/sites-enabled/default
	...
	location /static/ {
		root /home/www/project/<repository_name>;
	}
	location /media/ {
		root /home/www/project/<repository_name>;
	}
	...
```
Перезапускаем nginx-клиент
```
sudo service nginx restart
```
Теперь `staticfiles` и `mediafiles` будут проксироваться.\
Если у вас этого не происходит, то посмотрите лог ошибок nginx.
```
vim logs/nginx_errors.log
```
