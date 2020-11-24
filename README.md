# Инструкция по настройке Ubuntu-сервера для Django ORM
Данное руководство поможет вам настроить Ubuntu-server для запуска Django проектов в боевом режиме! Приятного прочтения.


## Обновляем пакеты и пакетный менеджер. Устанавливаем полезные утилиты.
```
sudo apt-get update
```
```
sudo apt-get install -y vim htop tree git curl zsh tmux nginx python3.8 python3-pip python3-venv
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
Добавляем шорткаты для удобный и продуктивной работы с терминалом
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
Создаем директорию, переходим в неё и клонируем репозиторий с проектом
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
## Настраиваем Gunicorn
Устанавливаем `gunicorn` и сохраняем список установленных пакетов
```
pip3 install gunicorn
pip3 freeze > requirements.txt
```
Создаем Gunicorn-конфиг
```
vim /home/www/project/<repository_name>/gunicorn_config.py
```
Вставляем параметры gunicorn'а
```
command = '/home/www/project/venv/bin/gunicorn'
pythonpath = '/home/www/project/<repository_name>/<settings_folder_name>'
bind = '127.0.0.1:8001'
workers = <(2*Количество_ядер)+1>
user = 'www'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=<settings_folder_name>.settings'
```
Создаем bash-скрипт для запуска gunicron'а
```
mkdir bin
vim /home/www/project/bin/start_gunicorn.sh
    #!/bin/bash
	source /home/www/project/venv/bin/activate
	exec gunicorn  -c "/home/www/project/<repository_name>/gunicorn_config.py" <settings_folder_name>.wsgi
```

