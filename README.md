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


## Модели
```python
# my_book_app/models.py
from django.db import models

class Book(models.Model):
	title = models.CharField(max_length=100)

class Goblin(models.Model):
    id = models.AutoField(primary_key=True)
    name = models.CharField(max_length=100),
    damage = models.IntegerField()
    hp = models.IntegerField()
    home = models.ForeignKey(GoblinCave, on_delete=models.CASCADE)



```
- Создание миграции  
**python manage.py makemigrations \<name\>**  
- Применение миграции к базе
**python manage.py migrate**
