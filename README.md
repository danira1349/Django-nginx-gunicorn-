# Django-nginx-gunicorn-
Django + nginx + gunicorn гайд по деплою для начинающих

Всем привет, я изучаю django и дошел до деплоя проекта, в сети не нашел гайда на русском, который помог бы мне полностью
шаг за шагом, провести эту процедуру. Поэтому пришлось использовать несколько разных статей и свести все воедино.
Чтобы таким же новичкам как я, не пришлось тратить несколько дней на деплой, решил собрать все в одном гайде,
так же и самому будет удобно освежить память, надеюсь он кому-нибудь поможет.
Я еще не сталкивался с рабочими проектами, т.к. только учусь, поэтому на исключительную правильность моих действий не претендую
По мере изучения и развития, буду апдейтить гайд


Я использовал amazon vps, там есть подробный гайд по регистрации и по созданию инстанса.
В качестве ОС выбираем ubuntu, создаем и скачиваем пару ключей, в процессе подключения инстанса
Так же, для тех кто использует amazon нужно открыть http порт, по умолчанию открыты только ssh. Заходим в меню наших инстансов, выбираем нужный, переходим по самой последней ссылке в параметрах инстанса "Security Groups", в нижнем меню выбираем "Inbound", далее "Edit", "Add", выбираем "HTTP" и жмем "Save", так же вы можете открывать другие порты
 
Далее заходим по ssh с этими ключами(у меня на пк стоит убунту, для запуска через винду на амазоне есть гайд)

`ssh -v -i ваш_ключ.pem ubuntu@ваш_айпи`    #айпи указан в настройках инстанса
жмем "y" и мы попадаем на наш инстанс

Апгрейдим систему
`sudo apt-get update`
`sudo apt-get upgrade`

Устанавливаем нужные инструменты  
`sudo apt-get install python3`  
`sudo apt-get install python3-setuptools`  
`sudo apt-get install nginx`  

Далее я все манипуляции производил в `/home/ubuntu`
Устанавливаем виртуальное окружение с python3  
`virtualenv --python=python3 --always-copy venv`
`source venv/bin/activate`

Устанавлием gunicorn
`pip install gunicorn`

Устанавливаем базу данных(postgresql)  

`sudo apt-get install -y postgresql postgresql-contrib libpq-dev python3-dev postgresql-server-dev-all`

Создаем пользователя и базу для нашего проекта
`sudo su - postgres`  
`createuser --interactive -P`  
`Enter name of role to add: db_user`  
`Enter password for new role:`  
`Enter it again:`  
`Shall the new role be a superdjuser? (y/n) n`  
`Shall the new role be allowed to create databases? (y/n) n`  
`Shall the new role be allowed to create more new roles? (y/n) n`  
`createdb --owner db_user db_name`  
`logout`  

Далее нам нужно клонировать проект в инстанс. Я предварительно загрузил проект на гитхаб и клонировал в директорию `/home/ubuntu`  
`sudo apt-get install git`  
`git clone https://github.com/ваш_юзер/ваш_проект.git`  # ссылка так же будет в корне вашего проекта на гитхабе

Устанавливаем зависимости 
`pip install -r requirements.txt`  
Если в вашем проекте нет этого файла, создайте его и положите в репозиторий, в нем отображаются все нужные инструменты которые используются в вашем проекте
Для создания в вашем окружении на локальной машине вводите: pip freeze > requirements.txt

Апдейтим `settings.py`  
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'db_name',
        'USER': 'db_user',
        'PASSWORD': 'ваш_пароль',
        'HOST': 'localhost',
        'PORT': '',
    },
}`
```
```
STATIC_URL = '/static/'  
STATIC_ROOT = os.path.join(BASE_DIR, "static_root")     #в эту папку скопируется вся статика, после команды 'collectstatic' и ее будем указывать в настройках nginx)  	
STATICFILES_DIRS = (os.path.join(BASE_DIR, "static"), )		
MEDIA_ROOT = os.path.join(BASE_DIR, "media")
MEDIA_URL = '/media/'  
```
удаляем все миграции из наших приложений, если они есть и создаем таблицы для каждого приложения  
`python manage.py makemigrations ваше_приложение`  # и так для каждого приложения

После этого  
`python manage.py migrate`  
`python manage.py createsuperuser` #стадартное создание админа  

создаем статик рут папку
`python manage.py collectstatic`

Теперь настраиваем nginx:  
`sudo vim /etc/nginx/sites-available/default`   # можно использовать любой удобный редактор, я привык к vim

Записываем в этот файл:  
```
server {
    listen 80;
    server_name ваш_внешний_ip_инстанса;
    access_log  /var/log/nginx/example.log;
    
    location /media  {
        alias /home/ubuntu/Project/project/media;	  #Указывайте ваш путь до папок меди и статик рут, чтобы узнать полный                                                        путь до каталога, перейдите в него и введите "pwd"
    }
    
    location /static {
        alias /home/ubuntu/Project/project/static_root;
    }
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $server_name;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
перезапускаем nginx  
`sudo service nginx restart`

заходим в папку проекта и запускаем gunicorn  
`gunicorn project.wsgi:application`   			#напоминаю, "project" - папка проекта, у вас может быть другое название

проверяем работу в браузере, вводим ip вашего инстанса

Теперь нам нужно автоматизировать работу сервера, чтобы он запускался автоматически после ребута инстанса, для этого нам понадобится supervisor  
`sudo apt-get install supervisor`  
`sudo vim /etc/supervisor/conf.d/project.conf`

Записываем в файл `project.conf`  
```
[program:project]
command=/home/ubuntu/venv/bin/gunicorn --bind localhost:8000 project.wsgi:application     #Так же проверьте путь до                                                                                                  #каталога и имя 	вашего проекта
enviroment=PYTHONPATH=/home/ubuntu/venv/bin
directory=/home/ubuntu/Diabetes_project/project
user=ubuntu  
```
Управление supervisor 
```
sudo supervisorctl reload                   #при последующем использовании сервера, часто будете использововать reload,
                                            #для перезагрузки и внесения изменений
sudo supervisorctl status                   #статус процесса
sudo supervisorctl reread                   #заного считать файл конфигурации
sudo supervisorctl update                   #пишем после добавления нового процесса в конфигурацию
sudo supervisorctl start django_project     #запустить проект, указанный в конфиге, у вас может быть другое название
```

Теперь можете перезагрузить инстанс и проверить работу supervisor
`sudo reboot`

Так же, настройки которые необходимо произвести для боевого сервера
Добавляем `404` и `500` страницы
создаем в папке темплейтов `404.html` и `500.html`, форматируем по желанию
Далее во `views` который лежит в основной папке проекта, рядом с `settings` добавляем следующие функции
```
from django.shortcuts import render_to_response
from django.template import RequestContext

def handler404(request):
    response = render_to_response('404.html', {},
                                  context_instance=RequestContext(request))
    response.status_code = 404
    return response

def handler500(request):
    response = render_to_response('500.html', {},
                                  context_instance=RequestContext(request))
    response.status_code = 500
    return response
```  

В `settings.py` меняем параметр `DEBUG = False`, на `DEBUG = True`  
и добавляем `ALLOWED_HOSTS = ['instance_public_ip', 'instance_public_dns']`  #public_ip и public_dnc указаны в настройках инстансов



