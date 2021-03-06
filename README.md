# djolar
🍕 A simple and light weight model search module for django (only 250+ lines of code), easy to connect front-end with backend

Why we need djolar
--------------

Performing search on the django model is little bit difficult when front-end needs dynamic and flexible search function.

Consider the book and author case, suppose we have model definition `Book`, and `Author` as below:

```python
class Author(models.Model):
    name = models.CharField('name of the author', max_length=50)
    age = models.IntegerField('age')

class Book(models.Model):
    name = models.CharField('name of the book', max_length=100)
    publish_at = models.DateTimeField('publish date')
    author = models.ForeignKey(Author)
    status = models.CharField('status of the book', max_length=10)
    createDate = models.DateTimeField('create at', auto_now=True)
```

And we need to implement a search engine for `Book` model in the front-end. Search criteria requirement would be like this:

* search by book name (contains)
* search by book name (exactly match)
* search by author name
* search by the book's author's age
* search by the book publish date range
* search critiera with the combination above.

So how can we meet the requirement above in django? You may begin thinking using serveral `filter` chain to perform the search, and the code may look this this:

```python
queryset = Book.objects.all()

if request.GET.get('name'):
    queryset = queryset.filter(name=request.GET['name'])

if request.GET.get('author'):
    queryset = queryset.filter(author__name=request.GET['author'])

if request.GET.get('age'):
    queryset = queryset.filter(author__name=request.GET['age'])

if request.GET.get('from'):
    queryset = queryset.filter(publish_at__gte=request.GET['from'])
    
if request.GET.get('to'):
    queryset = queryset.filter(publish_at__lte=request.GET['to'])

return queryset
```

Wow... it is really complicated.. But hold on, how can you support `exactly match` with book name? How did you write the query string? It is really a problem, isn't it?

So it is `djolar` show time....

Usage
--------

`djolar` make heavily use of DJANGO model `Q` object to build a compilicated filter. `Q` is very flexible for dynamic search on model. `djolar` is trying to help you convert the query string pass from front-end to a `Q` object. So that you can chain more filter criteria to the parsed `Q` object or apply the `Q` object directly to the model `filter` function.

To use `djolar`, you just need to do two things:

1. create a subclass of `DjangoSearchParser`. And defining the front-end query param field name and model field name mapping.
2. use correct `djolar` syntax to write a custom query param string.

Query param syntax
-----------------

All query field name value pair should be encoded and assign to `q` field name as below:

    q=key1:value1+key2:value2+key3:value3&s=order&extrafield=value
        
The above string indicate we need to search with 3 name value pairs:

```
(key1, value1)
(key2, value2)
(key3, value3)
```

The search syntax would be like this:

1. contains                 =>  key__co__value
                            * sql equal: `key like '%value%'`
2. exactly match            =>  key__eq__value, url encoded: key%3A%22value%22
                            * sql equal: `key = 'value'`
3. `in` operator            =>  key__in__[value1,value2,value3]
                            * sql equal:`key in (value1,value2,value3)`
4. `not` operator(AND)      =>  key__ni__[value1,value2]
                            * sql equal: `key not in (value1,value2)`
5. less than (lt)           =>  key__lt__value
                            * sql equal: `key < value`
6. less than or equal(lte) => key__lte__value
                            * sql equal: `key <= value`
7. great than (gt)          =>  key__gt__value
                            * sql equal: `key > value`
8. great than or equal(lte) => key__gte__value
                            * sql equal: `key >= value`

Example
-------

Firstly, we just need to subclass `DjangoSearchParser` to define a book search parser to help `djolar` to do query param and model field name mapping. The mapping is useful when you don't want to expose the django model field name to the front-end.

Suppose, the query param from front-end contains the following field names:

* `name` represent the name of the book
* `author` represent the author name of the book
* `from` represent the publish date range begin date
* `to` represent the publish date range end date

So the parser may look like this:

```python

from parser import DjangoSearchParser

class BookSearchParser(DjangoSearchParser):
    query_mapping = {
        'name': 'name',
        'age': 'author__age',
        'author': 'author__name',
        'from': 'publish_at__gte',
        'to': 'publish_at__lte',
    }
```

Then we can use the `BookSearchParser` to parse the query string send from front-end:

```python
import urllib

# Search by book name CONTAINS Programming, and the author name CONTAINS Dennis Ritchie
# We need to encode the param to ensure no conflict.
queryParam = '?q=' + urllib.quote('name:Programming+author:Dennis Ritchie')
queryParam = '?q=name%3AThe%20C%20Programming%20Language%2Bauthor%3ADennis%20Ritchie'

# Create a parser
parser = BookSearchParser()

# Parse the queryParam, and get an `Q` object
queryQ = parser.get_query_fields(queryParam)

# Perform filter
results = Book.objects.filter(queryQ)
```

Now you can change the queryParam as you need to perform more complicated search

```python

>>> queryParam = '?q=' + urllib.quote('name:Programming+author:Dennis Ritchie+from:2016-01-01+to:2016-12-31')
>>> print queryParam
'?q=name%3AProgramming%2Bauthor%3ADennis%20Ritchie%2Bfrom%3A2016-01-01%2Bto%3A2016-12-31'

# Parse
queryQ = parser.get_query_fields(queryParam)

# Search 
# Result with: 
#    1. Book name contains 'Programming', AND
#    2. author name contains 'Dennis Ritchie' AND
#    3. publish from 2016-01-01 to 2016-12-31
results = Book.objects.filter(queryQ)
```

More examples
------------

Contains operator, like the SQL `LIKE` concept

```python
# Book name contains python (case ignore)
queryQ = searcher.get_query_fields(QueryDict('q=name__eq__Python'))
```

Exactly match operator, like SQL `=`

```python
# Book name equal to 'Python'
queryQ = searcher.get_query_fields(QueryDict('q=name__eq__Python'))
```

IN operator, like SQL `in`

```python
# Book name in one of these values ('Python', 'Ruby', 'Swift')
queryQ = searcher.get_query_fields(QueryDict('q=name__in__[Python,Ruby,Swift]'))
```

NOT operator, LIKE SQL `NOT IN`

```python
# Book name NOT in these values ('Python', 'Ruby', 'Swift')
queryQ = searcher.get_query_fields(QueryDict('q=name__ni__[Python,Ruby,Swift]'))
```

Less than, LIKE SQL `<`

```python
# Author age less than 18
queryQ = searcher.get_query_fields(QueryDict('q=age__lt__18'))
```

Less than or equal to, LIKE SQL `<=`

```python
# Author age less or equal to 18
queryQ = searcher.get_query_fields(QueryDict('q=age__lte__18'))
```


Greater than, LIKE SQL `>`

```python
# Author age greater than 18
queryQ = searcher.get_query_fields(QueryDict('q=age__gt__18'))
```

Greater than or equal to, LIKE SQL `>=`

```python
# Author age greater than or equal to 18
queryQ = searcher.get_query_fields(QueryDict('q=age__gte__18'))
```

After getting the queryQ object, you can filter the Model or extend it more as you need

```python
Book.objects.filter(queryQ)

Book.objects.filter(queryQ).filter(pk__gte=1)

Book.objects.filter(queryQ | Q(pk__gte=1))
```


#### Integrate with Django & DJANGO RESET FRAMEWORK


```python

class APIBookSearcher(DjangoSearchParser):
    query_mapping = {
        'from': 'createDate__gte',
        'to': 'createDate__lte',
    }

    force_search = {
        'status__in': ('published', 'in progress')
    }

    # Default to filter 7 days sales
    default_search = {
        'createDate__range': (timedelta(days=-7) + now(), now())
    }
    

class APIBookListView(mixins.ListModelMixin, 
                      DjangoSearchMixin,
                      generics.GenericAPIView):
    '''
    Book list API
    '''
    serializer_class = APIReportXYNumericDataSerializer
    searcher_class = APIBookSearcher
    
    def get_search_queryset(self, *args, **kwargs):
        # Get origin queryset, the result will be process by djolar later
        return Order.objects.all()
```

Left the search thing to `djolar`, and go for a drink 🍻🍺☕️🍹 now...
