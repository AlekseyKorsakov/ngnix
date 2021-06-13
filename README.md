## Курсовая работа  
### Корсаков А. M30-312Б-18.  
  
# Постановка задачи:  
1. Скачать и установить веб-сервер.  
2. Настроить его на работу с localhost  
3. Реализовать форму с загрузкой файла  
3.1 Захостить python приложение из предыдущего семестра, при загрузке снимка рисовать в веб карту NDVI.  
  
# Ход выполнения работы:  
Работа выполнена на CentOS7.  
Создадим user "lesik" и зайдем в него.  
Директория проекта: "mkdir ~/project/"  
  
## Шаг 1. Установка необходимых компонентов.  
Установка репозитория EPEL:    

    [lesik@localhost project]$ sudo yum install epel-release

Установка pip - диспетчера пакетов Python, файлов разработки Python, Nginx.  

    [lesik@localhost project]$ sudo yum install python-pip python-devel gcc nginx  
  
## Шаг 2. Создание виртуальной среды Python.  
Установка virtualenv:  

    [lesik@localhost project]$ sudo pip install virtualenv  
Создание виртуальной среды:  

    [lesik@localhost project]$ virtualenv projectvenv  
Зайдем в виртальную среду:  

    [lesik@localhost project]$ source projectvenv/bin/activate  
Убедимся что мы начали работу в виртуальной среде. Ввод терминала выглядит следующим образом:  

    (projectvenv) [м@localhost project]$  
  
## Шаг 3. Создание и настройка приложения Flask.  
Установка uwsgi и flask:  

    (projectvenv) [lesik@localhost project]$ pip install uwsgi flask  
Создадим Flask:  

    (projectvenv) [lesik@localhost project]$ vi ~/project/app.py  
Исходный код: app.py   
 
Запуск приложения в фоновом режиме:  

    (projectvenv) [lesik@localhost project]$ python ~/project/app.py &  
> *Serving Flask app 'app' (lazy loading)  
> *Environment: prodction  
> *Debug mode: on  
> *Running on all addresses.  
> *Running on http://10.0.15:5000 (Press CTRL+C) to quit)  

Выведем первые 8 строчек html страницы: 

    (projectenv) [lesik@localhost project]$ curl -L http://10.0.2.15:5000 | head -n 8  
    
> \<!DOCTYPE html>  
>  \<html lang="ru">  
>  \<head>  
>   \<meta charset="UTF-8">  
>   \<title>Image processing</title>  
>  \</head>  
> \<body>  
>   \<p class="fadein1">Загрузите изображения для красного и инфракрасного спектров в формате tif:</p>  
      
Остановка Flask с помощью fg:  

    (projectvenv) [lesik@localhost project]$ fg  
    python app.py  
    ^C  
  
## Шаг 3. Создание точки входа WSGI.  
Создадим файл wsgi.py:  

    (projectvenv) [lesik@localhost project]$ vi ~/project/wsgi.py  
Внутри напишем:  

    from project import app  
    if __name__ == "__main__":  
      app.run()  
  
## Шаг 4. Настройка конфигурации uWSGI.  
Протестируем uWSGI:  

    (projectvenv) [lesik@localhost project]$ uwsgi --socket 0.0.0.0:8000 --protocol=http -w wsgi &  
Убедимся что по указанному ранее адресу, но с портом 8000 находится содержимое html страницы.  

    (projectvenv) [lesik@localhost project]$ curl -L http://10.0.15:8000 | head -n 5  
    
> \<!DOCTYPE html>  
> \<html lang="ru">  
> \<head>  
> \<meta charset="UTF-8">  
> \<title>Image processing</title>  

После этого приостановим uwsgi:  

    (projectvenv) [lesik@localhost project]$ fg  
    uwsgi --socket 0.0.0.0:8000 --protocol=http -w wsgi
    ^C  
Выйдем из виртуальной среды командой deactivate:  

    (projectvenv) [lesik@localhost project]$ deactivate  
Создадим файл конфигурации uWSGI:  

    [lesik@localhost project]$ vi ~/project/project.ini  
Внутри напишем:  

    [uwsgi]  
    module = wsgi  
    master = true  
    processes = 3  
    socket = project.sock  
    chmod-socket = 660  
    vacuum = true  
где "module = wsgi" - исполняемый модуль, созданный ранее файл "wsgi.py";  
"master = true" - означает что uWSGI будет запускаться в главном режиме;  
"processes = 3" - будет иметь 3 рабочих процесса для обслуживания запросов;  
"socket = project.sock" - сокет, который будет использовать uWSGI;  
"chmod-socket = 660" - права на процесс uWSGI;  
"vacuum = true" - сокет будет очищен по завершении работы процесса;  
  
## Шаг 5. Создание файла модуля systemd.  
Создадим файл службы:  

    [lesik@localhost project]$ sudo vi /etc/systemd/system/project.service  
Внутри напишем:

    [Unit]  
    Description=uWSGI for project  
    After=network.target  
    [Service]  
    User=lesik 
    Group=nginx  
    WorkingDirectory=/home/lesik/project  
    Environment="PATH=/home/lesik/project/projectvenv/bin"  
    ExecStart=/home/lesik/project/projectvenv/bin/uwsgi --ini project.ini  
    [Install]  
    WantedBy=multi-user.target  
где "[Unit]", "[Service]", "[Install]" - заголовки разделов;  
"Description" - описание службы;  
"After" - цель, по достижении которой будет производиться запуск;  
"User" - пользователь;  
"Group" - группа;  
"WorkingDirectory" - рабочая директория в которой хранятся исполняемые файлы;  
"Environment" - директория виртуальной среды;  
"ExecStart" - команда для запуска процесса;  
"WantedBy" - когда запускаться службе.  
Запустим созданную службу:  

    [lesik@localhost project]$ sudo systemctl start project  
    [lesik@localhost project]$ sudo systemctl enable project  
  
## Шаг 6. Настройка Nginx.  
Теперь необходимо настроить Nginx для передачи веб-запросов в  сокет с использованием uWSGI протокола.  
Откроем файл конфигурации Nginx:  

    [lesik@localhost project]$ sudo vi /etc/nginx/nginx.conf  
Найдем блок server {} в теле http: 

    ...  
      http {  
      ...  
        include /etc/nginx/conf.d/*.conf;  
        *
        server {  
            listen 80 default_server;  
            }  
      ...  
      }  
    ...  
 
Выше него (*) создадим свой:  

    server {  
      listen 80;  
      server_name 10.0.2.15;  
      
      location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/lesik/project/project.sock;
    }
Закроем и сохраним файл.  
Добавим nginx пользователя в свою группу пользователей с помощью следующей команды:  

    [lesik@localhost project]$ sudo usermod -a -G lesik nginx  
Предоставим группе пользователей права на выполнение в домашнем каталоге:  

    [lesik@localhost project]$ chmod 710 /home/lesik  
Запуск Nginx:  

    [lesik@localhost project]$ sudo systemctl start nginx  
    [lesik@localhost project]$ sudo systemctl enable nginx  
  
# Вывод:  
В ходе выполнения курсовой работы было создано приложение Flask в виртуальной средe, позволяющее загружать файлы формата ".tif" и получать цветное изображение NDVI. 
Была создана и настроена точка входа WSGI, с помощью котрой любой сервер приложений, поддерживающий WSGI, мог взаимодействовать с приложением Flask. 
Была создана служба systemd для автоматического запуска uWSGI при загрузке системы. 
Также был настроен серверный блок Nginx. 

