# Инструкция по настройке Ubuntu-сервера для Django ORM
Данное руководство поможет вам настроить Ubuntu-server для запуска Django проектов в боевом режиме! Приятного прочтения.

## Добавляем нового пользователя. Настраиваем доступ к серверу.
Создаём пользователя `www` --> Придумываем пароль --> Данные заполнять необязательно (Full Name, Room Number, ..., other).\
Добавляем пользователя `www` в группу `sudo`, чтобы получить права администратора.\
Отсоединяемся от сервера.\
Копируем ssh-ключ на сервер для нового пользоветеля.\
Подключаемся к серверу под новым пользователем.

```
sudo adduser www
sudo adduser www sudo
exit
ssh-copy-id www@<адрес сервера>
ssh www@<адрес сервера>
```

## Обновляем пакеты и пакетный менеджер. Устанавливаем полезные утилиты.
```
sudo apt-get update
```
```
sudo apt-get install -y vim htop tree git curl zsh tmux nginx
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




