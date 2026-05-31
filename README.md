# Django-CheetSheet-Ru
## Команды
- Создание проекта  
**django-admin startproject \<name\>**  
- Создание домена  
**python manage.py startapp \<name\>**  
- Запуск сервера  
**python manage.py runserver \<ip:port\> || \<port\>**  
- Конфигурирование  
При создании новых view, добавляем их в urls и в settings   
**path(path, view, kwargs, name)**, name позволяет ссылаться на view из любого места в Django, из templates в том числе
