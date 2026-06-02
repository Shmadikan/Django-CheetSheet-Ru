Вот перевод раздела `prefetch_related()` из документации Django:

---

## `prefetch_related()`

`prefetch_related(*lookups)`

Возвращает `QuerySet`, который автоматически извлечёт связанные объекты одним пакетом для каждого из указанных lookup-ов.

Цель схожа с `select_related` — оба метода предназначены для устранения лавины запросов к базе данных при обращении к связанным объектам, — однако стратегия принципиально иная.

`select_related` работает через SQL JOIN, включая поля связанных объектов в `SELECT`. Именно поэтому он получает связанные объекты в том же запросе. Однако, чтобы избежать огромных результирующих наборов при объединении через «многие», `select_related` ограничен единственно-значными связями — внешними ключами и «один к одному».

`prefetch_related`, напротив, выполняет отдельный запрос для каждой связи, а «объединение» делает на стороне Python. Это позволяет ему предзагружать связи «многие ко многим», «многие к одному» и объекты `GenericRelation` — то, что невозможно сделать через `select_related`, — а также поддерживаемые `select_related` связи «внешний ключ» и «один к одному». Также поддерживается предзагрузка `GenericForeignKey`, однако для каждого `ContentType` необходимо передать queryset в параметре `querysets` объекта `GenericPrefetch`.

### Пример

Предположим, у вас такие модели:

```python
from django.db import models

class Topping(models.Model):
    name = models.CharField(max_length=30)

class Pizza(models.Model):
    name = models.CharField(max_length=50)
    toppings = models.ManyToManyField(Topping)

    def __str__(self):
        return "%s (%s)" % (
            self.name,
            ", ".join(topping.name for topping in self.toppings.all()),
        )
```

И вы выполняете:

```python
>>> Pizza.objects.all()
<QuerySet [<Pizza: Hawaiian (ham, pineapple)>, <Pizza: Seafood (prawns, smoked salmon)>, ...]>
```

Проблема в том, что при каждом вызове `Pizza.__str__()` запрашивается `self.toppings.all()`, а это означает отдельный запрос к базе данных для **каждой** пиццы в QuerySet.

С `prefetch_related` можно сократить это до двух запросов:

```python
>>> Pizza.objects.prefetch_related("toppings")
```

Теперь при каждом обращении к `self.toppings.all()` данные берутся из заранее заполненного кэша, а не из базы данных.

### Важные особенности

**Момент выполнения.** Дополнительные запросы `prefetch_related()` выполняются после того, как основной QuerySet начал вычисляться.

**Нет гарантии консистентности.** Нет механизма, предотвращающего изменение данных между выполнением основного запроса и дополнительных. Например, если пицца удалена после основного запроса, у неё не окажется начинок:

```python
>>> Pizza.objects.prefetch_related("toppings")
# "Hawaiian" была удалена в другой сессии
<QuerySet [<Pizza: Hawaiian ()>, <Pizza: Seafood (prawns, smoked salmon)>]>
```

**Загрузка в память.** Весь кэш основного QuerySet и всех указанных связанных объектов загружается в память целиком — в отличие от обычного поведения QuerySet.

**Цепочки методов со своими запросами сбрасывают кэш.** Если после `prefetch_related` вызвать метод, генерирующий новый запрос, кэш не поможет:

```python
>>> pizzas = Pizza.objects.prefetch_related("toppings")
>>> [list(pizza.toppings.filter(spicy=True)) for pizza in pizzas]
```

Здесь `pizza.toppings.filter()` — это новый запрос, а не `toppings.all()`. Предзагрузка не только не помогает, но и вредит производительности, так как добавляет лишний запрос.

**Изменение данных сбрасывает кэш.** При вызове `add()`, `create()`, `remove()`, `clear()` или `set()` на связанном менеджере кэш предзагрузки для этой связи очищается.

**Цепочки связей** через `__` работают так же, как при обычных запросах:

```python
>>> Restaurant.objects.prefetch_related("pizzas__toppings")
# 3 запроса: рестораны → пиццы → начинки

>>> Restaurant.objects.select_related("best_pizza").prefetch_related("best_pizza__toppings")
# 2 запроса: best_pizza уже загружена через select_related
```

**Сброс prefetch.** Чтобы отменить все ранее указанные предзагрузки, передайте `None`:

```python
>>> non_prefetched = qs.prefetch_related(None)
```

**Разделение объектов.** Объекты, созданные в ходе предзагрузки, могут использоваться совместно — один Python-экземпляр модели может появляться в нескольких местах дерева объектов. Как правило, это не проблема и даже экономит память и время CPU.

**`GenericForeignKey`.** Количество запросов зависит от данных: по одному запросу на каждую таблицу, на которую ссылается `GenericForeignKey`.

**Оператор `IN`.** В большинстве случаев `prefetch_related` использует SQL-запрос с оператором `IN`. Для больших QuerySet это может генерировать большой `IN`-список, что способно замедлить разбор или выполнение запроса. Всегда профилируйте под свой конкретный случай.

**Совместимость с `iterator()`.** Если вы используете `iterator()` для выполнения запроса, `prefetch_related()` будет работать только при указании значения `chunk_size`.

---

### Объект `Prefetch`

Для более тонкого управления предзагрузкой используйте объект `Prefetch`.

В простейшем виде `Prefetch` эквивалентен строковому lookup:

```python
>>> from django.db.models import Prefetch
>>> Restaurant.objects.prefetch_related(Prefetch("pizzas__toppings"))
```

Можно передать **собственный queryset** через аргумент `queryset`, например для изменения сортировки:

```python
>>> Restaurant.objects.prefetch_related(
...     Prefetch("pizzas__toppings", queryset=Toppings.objects.order_by("name"))
... )
```

Или сократить число запросов, добавив `select_related()`:

```python
>>> Pizza.objects.prefetch_related(
...     Prefetch("restaurants", queryset=Restaurant.objects.select_related("best_pizza"))
... )
```

Аргумент **`to_attr`** позволяет сохранить результат предзагрузки в произвольный атрибут (как список, а не кэш менеджера):

```python
>>> vegetarian_pizzas = Pizza.objects.filter(vegetarian=True)
>>> Restaurant.objects.prefetch_related(
...     Prefetch("pizzas", to_attr="menu"),
...     Prefetch("pizzas", queryset=vegetarian_pizzas, to_attr="vegetarian_menu"),
... )
```

Использование `to_attr` рекомендуется при фильтрации результатов предзагрузки — это однозначнее, чем хранение отфильтрованных результатов в кэше менеджера:

```python
# Рекомендуется:
restaurants = Restaurant.objects.prefetch_related(
    Prefetch("pizzas", queryset=queryset, to_attr="vegetarian_pizzas")
)
vegetarian_pizzas = restaurants[0].vegetarian_pizzas

# Не рекомендуется:
restaurants = Restaurant.objects.prefetch_related(
    Prefetch("pizzas", queryset=queryset),
)
vegetarian_pizzas = restaurants[0].pizzas.all()
```

**Несколько баз данных.** `Prefetch` учитывает выбор базы данных: если внутренний запрос не указывает БД, используется та же, что и у внешнего запроса.

### ⚠️ Порядок аргументов имеет значение

```python
# Нормально — "pizzas" избыточен, но не вредит:
prefetch_related("pizzas__toppings", "pizzas")

# ValueError — попытка переопределить queryset уже увиденного lookup:
prefetch_related("pizzas__toppings", Prefetch("pizzas", queryset=Pizza.objects.all()))

# AttributeError — "pizza_list" ещё не существует в момент обработки:
prefetch_related("pizza_list__toppings", Prefetch("pizzas", to_attr="pizza_list"))
```

Поэтому всегда тщательно продумывайте порядок аргументов `prefetch_related`.




Страница уже была загружена в прошлый раз — раздел `select_related` есть в полученном содержимом. Вот перевод:

---

## `select_related()`

`select_related(*fields)`

Возвращает `QuerySet`, который «следует» по связям внешних ключей, выбирая дополнительные данные связанных объектов при выполнении запроса. Это оптимизация производительности, которая приводит к одному более сложному запросу, но позволяет в дальнейшем обращаться к связанным объектам без дополнительных запросов к базе данных.

### Сравнение обычного запроса и `select_related`

Обычный запрос:

```python
# Обращение к БД
e = Entry.objects.get(id=5)

# Снова обращение к БД — за связанным объектом Blog
b = e.blog
```

С `select_related`:

```python
# Обращение к БД
e = Entry.objects.select_related("blog").get(id=5)

# Не обращается к БД — e.blog уже заполнен из предыдущего запроса
b = e.blog
```

Использование с любым QuerySet:

```python
from django.utils import timezone

blogs = set()
for e in Entry.objects.filter(pub_date__gt=timezone.now()).select_related("blog"):
    # Без select_related() каждая итерация делала бы отдельный запрос к БД
    blogs.add(e.blog)
```

Порядок вызова `filter()` и `select_related()` не важен — оба варианта равнозначны:

```python
Entry.objects.filter(pub_date__gt=timezone.now()).select_related("blog")
Entry.objects.select_related("blog").filter(pub_date__gt=timezone.now())
```

### Цепочки связей через `__`

Можно следовать по цепочке внешних ключей так же, как при обычных запросах. Например, для таких моделей:

```python
class City(models.Model):
    pass

class Person(models.Model):
    hometown = models.ForeignKey(City, on_delete=models.SET_NULL, blank=True, null=True)

class Book(models.Model):
    author = models.ForeignKey(Person, on_delete=models.CASCADE)
```

Запрос `Book.objects.select_related('author__hometown').get(id=4)` загрузит сразу и связанный `Person`, и связанный `City`:

```python
b = Book.objects.select_related("author__hometown").get(id=4)
p = b.author      # нет запроса к БД
c = p.hometown    # нет запроса к БД

# Без select_related:
b = Book.objects.get(id=4)  # запрос к БД
p = b.author                # запрос к БД
c = p.hometown              # запрос к БД
```

В список полей можно передавать любые связи `ForeignKey` и `OneToOneField`, в том числе **обратные** `OneToOneField` — вместо имени поля используется `related_name`.

### Вызов без аргументов

Можно вызвать `select_related()` без аргументов — тогда Django автоматически последует по всем ненулевым внешним ключам, которые найдёт. Nullable-ключи нужно указывать явно. Такой подход **не рекомендуется** в большинстве случаев — итоговый запрос скорее всего окажется сложнее и вернёт больше данных, чем реально нужно.

### Сброс списка связей

Чтобы очистить все ранее указанные связи, передайте `None`:

```python
>>> without_relations = queryset.select_related(None)
```

### Цепочки вызовов

Несколько вызовов `select_related` суммируются — запись `select_related('foo', 'bar')` равнозначна `select_related('foo').select_related('bar')`.
