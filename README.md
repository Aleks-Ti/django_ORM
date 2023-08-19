# Django ORM Примеры запросов и возможности

## Перевод статьи

Оригинал

```python
git@github.com:Jdsleppy/django-orm-cheatsheet.git
```

Если бы вы хотели прочитать много текста, вы бы читали документацию по Django, особенно эти страницы:

- [Queryset API (select_related, bulk_update, etc.)](https://docs.djangoproject.com/en/dev/ref/models/querysets/)
- [Query Expressions (Subquery, F, annotate, aggregate, etc.)](https://docs.djangoproject.com/en/dev/ref/models/expressions/)
- [Aggregation](https://docs.djangoproject.com/en/dev/topics/db/aggregation/)
- [Database Functions](https://docs.djangoproject.com/en/dev/ref/models/database-functions/)

Но вы этого не хотите, поэтому вы здесь. Я перейду к делу.

# Модели для примеров

```python
from datetime import date

from django.db import models


class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()

    def __str__(self):
        return self.name


class Author(models.Model):
    name = models.CharField(max_length=200)
    email = models.EmailField()

    def __str__(self):
        return self.name


class Entry(models.Model):
    blog = models.ForeignKey(Blog, on_delete=models.CASCADE)
    headline = models.CharField(max_length=255)
    body_text = models.TextField()
    pub_date = models.DateField()
    mod_date = models.DateField(default=date.today)
    authors = models.ManyToManyField(Author)
    number_of_comments = models.IntegerField(default=0)
    number_of_pingbacks = models.IntegerField(default=0)
    rating = models.IntegerField(default=5)

    def __str__(self):
        return self.headline
```

# Starting point

```python
Entry.objects.all()

# SELECT "blog_entry"."id", "blog_entry"."blog_id", ...
# FROM "blog_entry" ...;

# [<Entry: Off foot official kitchen another turn.>,
#  <Entry: Yet year marriage yes her she.>,
#  <Entry: Special mission in who son sort.>,
#  ...
# ]
```

# Делайте меньше запросов

Это самый выгодный вариант. Используйте их чаще.

## `select_related()`

\*-to-one relationships (one-to-one, many-to-one)

### without `select_related()`

```python
entry = Entry.objects.first()
# SELECT ... FROM "blog_entry" ...;

# при доступе к этому атрибуту выполняется второй запрос
blog = entry.blog
# SELECT ... FROM "blog_blog" WHERE ...;
```

### with `select_related()`

```python
entry = Entry.objects.select_related("blog").first()
# SELECT "blog_entry"."id", ... "blog_blog"."id", ...
# FROM "blog_entry"
# INNER JOIN "blog_blog" ...;

blog = entry.blog
# запрос не выполняется, поскольку мы объединились с таблицей blog выше
```

## `prefetch_related()`

\*-to-many relationships (one-to-many, many-to-many)

### without `prefetch_related()`

```python
entry = Entry.objects.first()
# SELECT ... FROM "blog_entry" ...;

# этот связанный запрос снова попадает в базу данных
authors = list(entry.authors.all())
# SELECT ...
# FROM "blog_author" INNER JOIN "blog_entry_authors"...
# WHERE "blog_entry_authors"."entry_id" = 4137;
```

### with `prefetch_related()`

```python
entry = Entry.objects.prefetch_related("authors").first()
# SELECT ... FROM "blog_entry" ...;
# SELECT ...
# FROM "blog_author" INNER JOIN "blog_entry_authors" ...
# WHERE "blog_entry_authors"."entry_id" IN (4137);

authors = list(entry.authors.all())
# запрос не выполняется, так как у нас есть in-memory
# поиск соответствующих авторов, указанных выше
```

## `update()`

### without `update()`

```python
david = (
    Author.objects.filter(name__startswith="David")
    .prefetch_related("entry_set")
    .first()
)
# SELECT ... FROM "blog_author" WHERE "blog_author"."name" LIKE 'David%' ...;
# SELECT ... FROM "blog_entry" INNER JOIN "blog_entry_authors" ...;

# Существует множество неэффективных способов обновления всех записей Дэвида.
# Это один из них.
for entry in david.entry_set.all():
    entry.rating = 5
    entry.save()
# Один запрос для каждой записи. Даже если мы используем bulk_update(),
# мы все равно делаем больше запросов, чем нам нужно.
```

### with `update()`

```python
Entry.objects.filter(authors__name__startswith="David").update(rating=5)
# UPDATE "blog_entry" SET "rating" = 5 WHERE "blog_entry"."id" IN ...;
```

# Запрашивать не только колонки, но и другие объекты, т.е. работать в БД, а не в Python

## `annotate()` (первый взгляд)

`annotate()` добавляет атрибут в каждую строку результата. Используйте эту функцию вместе с `F()` для добавления одного поля из связанного объекта (это имеет смысл, когда объекты разделены длинной цепочкой связей).

`annotate()` является очень мощным и имеет смысл использовать во многих ситуациях, поэтому просто потерпите этот первый нереалистичный пример.

```python
entries = Entry.objects.annotate(blog_name=F("blog__name"))[:5]
# SELECT "blog_entry"."id", ... "blog_blog"."name" AS "blog_name"
# FROM "blog_entry" INNER JOIN "blog_blog" ...;

# Теперь у нас есть объекты Entry с одним дополнительным атрибутом: blog_name
[entry.blog_name for entry in entries]
# ['Hunter-Rhodes',
#  'Mcneil PLC',
#  'Banks, Hicks and Carpenter',
#  'Anderson PLC',
#  'George-Bray']
```

## `filter()` and `Q()`

Используйте `Q()`, чтобы больше фильтровать в базе данных и меньше в приложении.

```python
low_engagement_posts = Entry.objects.filter(
    Q(number_of_comments__lt=20) | Q(number_of_pingbacks__lt=20)
)

list(low_engagement_posts)
# SELECT ... FROM "blog_entry"
# WHERE
#   ("blog_entry"."number_of_comments" < 20 OR
#   "blog_entry"."number_of_pingbacks" < 20);
```

## Query запросы

Many parts of the Django ORM expect a [query expression](https://docs.djangoproject.com/en/dev/ref/models/expressions/). Query expressions include:

- references to fields (possibly on related objects) `F()`
- SQL functions like `CASE`, `NOW`, etc.
- Subqueries with `Subquery()`

### `F()`

Классическим примером является инкремент, но напомним, что `F()` может использоваться везде, где требуется выражение запроса.

```python
Entry.objects.filter(authors__name__startswith="David").first().rating
# 5

Entry.objects.filter(authors__name__startswith="David").update(
    rating=F("rating") - 1
)
# UPDATE "blog_entry"
#   SET "rating" = ("blog_entry"."rating" - 1) WHERE ...;

Entry.objects.filter(authors__name__startswith="David").first().rating
# 4
```

### `CASE` and conditional logic

[Django docs page](https://docs.djangoproject.com/en/dev/ref/models/conditional-expressions/)

`Value()` Ниже приведено выражение запроса для строкового, целого, bool и т.д. литерального значения. В качестве значения `then` или даже вместо условия `rating=` можно использовать `F()` или другое выражение.

```python
entries = Entry.objects.annotate(
    coolness=Case(
        When(rating=5, then=Value("super cool")),
        When(rating=4, then=Value("pretty cool")),
        default=Value("not cool"),
    )
)
# SELECT ...
# CASE
#   WHEN "blog_entry"."rating" = 5 THEN 'super cool'
#   WHEN "blog_entry"."rating" = 4 THEN 'pretty cool'
#   ELSE 'not cool'
# END AS "coolness"
# FROM "blog_entry" LIMIT 5;

[f"Entry {e.pk} is {e.coolness}" for e in entries[:5]]
# ['Entry 4137 is super cool',
#  'Entry 4138 is not cool',
#  'Entry 4139 is not cool',
#  'Entry 4140 is pretty cool',
#  'Entry 4141 is not cool']
```

### `Subquery()` and `OuterRef()`

Аннотируйте каждый блог заголовком самой последней записи.

Этот паттерн - единственное применение `Subquery()`, которое мне удалось найти: запрос отношения \*-ко-многим, `OuterRef("pk")` для "объединения" строк, использование `values()` для возврата одного столбца и `[:1]` для возврата одной строки из подзапроса. Это практически копия примера из документации Django.

```python
blogs = Blog.objects.annotate(
    most_recent_headline=Subquery(
        Entry.objects.filter(blog=OuterRef("pk"))
        .order_by("-pub_date")
        .values("headline")[:1]
    )
)
[(blog, str(blog.most_recent_headline)) for blog in blogs[:5]]
# SELECT
#   "blog_blog"."id",
#   "blog_blog"."name",
#   "blog_blog"."tagline",
#   (
#     SELECT
#       U0."headline"
#     FROM
#       "blog_entry" U0
#     WHERE
#       U0."blog_id" = ("blog_blog"."id")
#     ORDER BY
#       U0."pub_date" DESC
#     LIMIT
#       1
#   ) AS "most_recent_headline"
# FROM
#   "blog_blog"
# LIMIT
#   5;


# [(<Blog: Robinson-Wilson>, 'Three space maintain subject much.'),
#  (<Blog: Anderson PLC>, 'Rock authority enjoy hundred reduce behavior.'),
#  (<Blog: Mcneil PLC>, 'Visit beyond base.'),
#  (<Blog: Smith, Baker and Rodriguez>, 'Tree look culture minute affect.'),
#  (<Blog: George-Bray>, 'Then produce tree quality top similar.')]
```

# Select less data

## `values()` (part 1)

Выберите только указанные поля, как словари. Можно также выбрать аннотации, например, так:

```python
Entry.objects.annotate(num_authors=Count("authors")).values(
    "rating", "num_authors"
)[:5]
# SELECT
#   "blog_entry"."rating",
#   COUNT("blog_entry_authors"."author_id") AS "num_authors"
# FROM "blog_entry" LEFT OUTER JOIN "blog_entry_authors" ...;

# [{'num_authors': 3, 'rating': 1},
#  {'num_authors': 4, 'rating': 4},
#  {'num_authors': 4, 'rating': 2},
#  {'num_authors': 3, 'rating': 2},
#  {'num_authors': 3, 'rating': 5}]
```

## `exists()`

Если вам нужен ответ "да/нет", что что-то существует, это быстрее, чем получение всей строки.

```python
unrealistic_data_exists = Entry.objects.filter(
    mod_date__lt=F("pub_date")
).exists()
# SELECT
#  (1) AS "a"
#   FROM "blog_entry"
#   WHERE "blog_entry"."mod_date" < ("blog_entry"."pub_date")
#   LIMIT 1;

unrealistic_data_exists
# True
```

## `only()`

Аналогично `values()`, но вместо словаря вы получаете экземпляр модели. Однако следует иметь в виду, что при обращении к полям, которые не были получены изначально, будет выполнен дополнительный запрос к БД.

# Grouping data

## `values()` (part 2) and `GROUP BY`

The Django docs contain [this hidden gem](https://docs.djangoproject.com/en/dev/topics/db/aggregation/#values):

> "Обычно аннотации генерируются по каждому объекту - аннотированный QuerySet возвращает по одному результату для каждого объекта исходного QuerySet. Однако, когда для ограничения столбцов, возвращаемых в наборе результатов, используется предложение `values()`, исходные результаты группируются в соответствии с уникальными комбинациями полей, указанных в предложении `values()`. Затем для каждой уникальной группы выдается аннотация; аннотация вычисляется по всем членам группы".

```python
[
    (str(e["pub_date"]), e["avg_rating"])
    for e in Entry.objects.values("pub_date").annotate(avg_rating=Avg("rating"))
]
# SELECT
#   "blog_entry"."pub_date",
#   AVG("blog_entry"."rating") AS "avg_rating"
# FROM "blog_entry"
# GROUP BY "blog_entry"."pub_date";

# [
#  ('2022-04-07', 2.235294117647059),
#  ('2022-04-08', 2.8157894736842106),
#  ('2022-04-09', 2.0285714285714285),
#  ('2022-04-10', 2.96875),
#  ('2022-04-11', 2.3636363636363638),
#  ('2022-04-12', 2.725),
#  ('2022-04-13', 2.5),
#  ('2022-04-14', 2.761904761904762),
#  ...
# ]
```

## `aggregate()`

Подобно `annotate()`, но вместо добавления значения для каждой строки он сводит запрос к одной строке.
