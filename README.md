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

class GoblinCave(models.Model):
	cave_id = models.IntegerField()
	hp = models.IntegerField()

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

Также для вывода мы можем добавить __str__.  
Если у нас на одну модель (1) ссылается другая модель (2), то мы можем получить множество всех значений, которые на неё ссылаются:  
```python
q = GoblinCave.objects.get(pk=1) # pk псевдоним для primary_key
q.goblin_set.all() # Получить список всех связанных объектов
q.goblin_set.create(...) # создать связанный объект
```

## Админка
- Создать админа  
python manage.py createsuperuser  
- Добавление моделей в админ панель:
```python
admin.site.register(GoblinCave)
admin.site.register(Goblin)
```
Мы можем объявлять классы моделей админок и добавлять им нужные поля для конфигурирования:  
```python
list_display = ('id', 'doctor', 'post_processed_review', 'ip_address', 'user', 'review_date_time', 'original_review', 'get_speciality')
list_editable = ('post_processed_review', 'ip_address', 'user', 'doctor')
raw_id_fields = ('doctor',) # делает возможность редактирования через строку, а не select
readonly_fields = ('review_date_time', 'original_review')
search_fields = ('review_date_time', 'original_review', 'ip_address', 'doctor__name', 'user__username')
list_per_page = 50
```
Мы также можем объявлять свои столбцы, для этого нужно определить метод, и указать его в отображении:  
```python
list_display = ('id', 'doctor', 'post_processed_review', 'ip_address', 'user', 'review_date_time', 'original_review', 'get_speciality')
    list_editable = ('post_processed_review', 'ip_address', 'user', 'doctor')
    raw_id_fields = ('doctor',)
    readonly_fields = ('review_date_time', 'original_review')
    search_fields = ('review_date_time', 'original_review', 'ip_address', 'doctor__name', 'user__username')
    list_per_page = 50
```

Также естественным подходом является переопределение методов, и других полей, если мы хотим изменить стандартное поведение админки. К примеру:  
- get_queryset() - чтобы изменить с какими объектами будет работа
- formfield_overrides - dict, где каждому объекту сопоставляется какой у него виджет

## ORM
При использовании методов orm (например all), возвращается query_set, с отобранными по условию объектами.  
Можно использовать другие методы для фильтрации и нужной выборке:  
- filter  
Entry.objects.filter(pub_date__year=2006) - **kwargs аргументы  
- exclude
Entry.objects.exclude(**kwargs) - всё что не подошло под условик.
**Важно**: query_set возвращает query_set, так что можно делать цепочки вызовов.
При этом каждый отдельный query_set это отдельный экземпляр класса.
- get  
Позволяет вернуть ровно один объект, подходящий под условик
- all
Вернуть все объекты.
Также к query_set можно применять все операции аналогичные спискам (индексы, срезы)
## Lookups
В Django есть мощное соглашение, которое позволяет задавать дополнительные действия при поиске полей:  
Если поле в методе для ORM задать в формате field_name__**action**, то введя специальный action будет выполнение дополнительное условие отбора  
К примеру:
- name__startswith="ben" - пон
- name__endswith="lol"
Кроме условий, мы можем например делать таким образом неявный join:  
**GoblinCave.objects.filter(model_name__field_name=?)**, условие аналогично join с model_name и отбору по полю. Глубина неограничена, также join работает двусторонне




