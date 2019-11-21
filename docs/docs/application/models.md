
## Overview

Assembly supports RDBMS (Postgresql, MySQL, SQLite) via **Active-Alchemy**, a wrapper around **SQLAlchemy** that makes it simple to use your models in an active record like manner, while it still uses the SQLAlchemy `db.session` underneath. 

Assembly requires you to create your models in  `__models__.py`, in the View Package folder. `__models__.py` loads automatically. 

Models are classes that extends `db.Model` 

A simple Models would look like this

```
|- main/
    |- __init__.py
    |- __models__.py

```
 

```
# main/__models__.py

from assembly import db

class Article(db.Model):
    title = db.Column(db.String(255))
    content = db.Column(db.Text)
    published_at = db.Column(db.DateTime)
    image = db.Column(db.StorageObjectType)

```

in views, model can be accessed with `models.$ModelClassName`. 

```
# main/__init__.py

from assembly import Assembly, models

class Article(Assembly):

    def get(self, id):
        article = models.Article.get(id)
        return {
            "article": article
        }

```

Note: **Every model class created have a reference in the `models` object**. You can access any models in the application.



**Extension: <a href="https://github.com/mardix/active-alchemy" target="_blank">Active-Alchemy</a>**

Extension: <a href="https://www.sqlalchemy.org/" target="_blank">SQLAlchemy</a>

---

## Features
- Automatically creates and manages the database connection
- Supports Postgresql, MySQL, SQLite
- Each model is a Python class that subclasses assembly.db.Model
- Each attribute of the model represents a database field by using db.Column
- With all of this, Assembly gives you an automatically-generated database-access API.
- ActiveAlchemy automatically creates the session, model and everything necessary for SQLAlchemy.
- It provides easy methods such as `query()`, `create()`, `update()`, `delete()`,
 to select, create, update, delete entries respectively.   
- It automatically create a primary key for your table
- It adds the following columns: `id`, `created_at`, `updated_at`, `is_deleted`, `deleted_at`
- When `delete()`, it soft deletes the entry so it doesn't get queried. But it still
exists in the database. This feature allows you to un-delete an entry
- It uses Arrow for DateTime
- DateTime is saved in UTC and uses the ArrowType from the SQLAlchemy-Utils
- Added some data types: JSONType, EmailType, and the whole SQLAlchemy-Utils Type
- db.now -> gives you the Arrow UTC type
- It is still SQLAlchemy. You can access all the SQLAlchemy awesomeness
---

## Configuration

Assembly, via Active-Alchemy comes, with a `PyMySQL` and `PG8000` as drivers for MySQL 
and PostgreSQL respectively, because they are in pure Python. But you can use 
other drivers for better performance. `SQLite` is already built in Python. 
  

In the `config.py` set **DB_URL** 


**DB_URL**

**DB_URL** follows RFC-1738, and usually can include username, password, hostname, database name as well as optional keyword arguments for additional configuration. In some cases a file path is accepted, and in others a “data source name” replaces the “host” and “database” portions. The typical form of a database URL is:

```
dialect+driver://username:password@host:port/database
```

**Postgresql**

```
DB_URL = "postgresql+pg8000://user:password@host:port/database"
```

**MySQL**

```
DB_URL = "mysql+pymysql://user:password@host:port/database"
```

**SQLite**

```
DB_URL = "sqlite:////database.db"
```

or in memory

```
DB_URL = "sqlite://"
```
---


## Model

Assembly, when **DB_URL** is set, will attempt to automatically connect to the database. Assembly exposes `db` which is an instance of `ActiveAlchemy`.

```
from assembly import db
```

To start, create a model class and extends it with `db.Model`

```
# main/__models__.py

from assembly import db

class Article(db.Model):
    title = db.Column(db.String(255))
    content = db.Column(db.Text)
    published_at = db.Column(db.DateTime)
    image = db.Column(db.StorageObjectType)
    
```

- Upon creation of the table, db.Model will add the following columns: `id`, `created_at`, `upated_at`, `is_deleted`, `deleted_at`

- It does an automatic table naming (if no table name is already defined using the `__tablename__` property)
by using the class name. So, for example, a `Article` model gets a table named `article`,  `User` becomes `user`, `TodoList` becomes `todo_list`
The name will not be plurialized.

### db.Model

**db.Model** extends your model with helpers that turn your model into an active record like model. But underneath, it still uses the ``db.session`` 

**db.Model** also adds a few preset columns on the table: 

``id``: The primary key

``created_at``: Datetime. It contains the creation date of the record

``updated_at``: Datetime. It is updated whenever the record is updated.

``deleted_at``: Datetime. Contains the datetime the record was soft-deleted. 

``is_deleted``: Boolean. A flag to set if record is soft-deleted or not

**Soft delete** marks a record as deleted so it doesn't get queried, but it still exists in the database. This allows you to undo a delete. If you want to completely delete an entry, you can set `$entry.delete(hard_delete=True)` to do so. 

---

### Create tables

Having created all your models in `__models__.py`, to create the tables, you need to use the CLI command.

```
asm-admin sync-models
```

It will automatically create all the models if they don't exist.

**Note** Whenever you are launching your application, make sure this command gets executed. Otherwise the tables will not exist.

---

### CRUD

Below are example of some CRUD operations you can do with your models.

#### query

`query(*args, **kwargs)`

To start querying the DB and returns a ``db.session.query`` object to filter or apply more conditions.

```
from assembly import models

for user in models.User.query():
    print(user.login)

```

By default `query()` will show only all non-soft-delete records. To display both deleted and non deleted items, add the arg: ``include_deleted=True``

```
for user in models.User.query(include_deleted=True):
    print(user.login)
```

To select columns...

```
for user in models.User.query(models.User.name.distinct(), models.User.location):
    print(user.login)
```

To use with filter...

```
all = models.User
        .query(models.User.name.distinct, models.User.location)
        .order_by(models.User.updated_at.desc())
        .filter(models.User.location == "Charlotte")
```

#### get

`get(id)`

Get one record by id. By default it will query only a record that is not soft-deleted

```
id = 1234
user = models.User.get(id)
print(user.id)
print(user.login)
```

To query a record that has been soft deleted, just set the argument ``include_deleted=True``

```
id = 234
user = models.User.get(id, include_deleted=True)
```
		
#### create

`create(**kwargs)`

To create/insert new record. Same as __init__, but just a shortcut to it.

```
record = models.User.create(login='abc', passw_hash='hash', profile_id=123)
print (record.login) # -> abc
```

or you can use the constructor with save()

```
record = models.User(login='abc', passw_hash='hash', profile_id=123).save()
print (record.login) # -> abc
```

or 

```
record = models.User(login='abc', passw_hash='hash', profile_id=123)
record.save()
print (record.login) # -> abc
```
	
#### update

`update(**kwargs)`

Update an existing record 

```
record = models.User.get(124)
record.update(login='new_login')
print (record.login) # -> new_login
```

#### delete

`delete()`

To soft delete a record. ``is_deleted`` will be set to True and ``deleted_at`` datetime will be set

```
record = models.User.get(124)
record.delete()
print (record.is_deleted) # -> True
```

To soft **UNdelete** a record. ``is_deleted`` will be set to False and ``deleted_at`` datetime will be None

```
record = models.User.get(124)
record.delete(hard_delete=False)
print (record.is_deleted) # -> False
```

To HARD delete a record. The record will be deleted completely

```
record = models.User.get(124)
record.delete(hard_delete=True)
```

#### save

`save()`

A shortcut to update an entry

```
record = models.User.get(124)
record.login = "Another one"
record.save()
```
---

#### Method Chaining 

For convenience, some method chaining are available

```
user = models.User(name="Mardix", location="Charlotte").save()

models.User.get(12345).update(location="Atlanta")

models.User.get(345).delete().delete(False).update(location="St. Louis")
```

---


#### Aggegated selects

```
class Product(db.Model):
    name = db.Column(db.String(250))
    price = db.Column(db.Numeric)
    
price_label = db.func.sum(models.Product.price).label('price')
results = models.Product.query(price_label)
```

**Learn more on <a href="https://github.com/mardix/active-alchemy" target="_blank">Active-Alchemy</a>**

---

## Views

Views can easily access your models, via the `models` object.

```
from assembly import Assembly, models
```

All the models created will have their reference in the `models` object.

```
# main/__init__.py

from assembly import Assembly, models

class Article(Assembly):

    def get(self, id):
        article = models.Article.get(id)
        return {
            "article": article
        }

```


Learn more on **[Views](../application/views.md) **

---

## Columns

Columns are also properties under `db` object.

ie:

```
from assembly import db

class Article(db.Model):
    title = db.Column(db.String(250))
    content = db.Column(db.Text)
    
```

### Column Types

Read mode about <a href="https://docs.sqlalchemy.org/en/13/core/type_basics.html" target="_blank">Column and Data Types</a>

- BIGINT
- BINARY
- BLOB
- BOOLEAN
- BigInteger
- Boolean
- CHAR
- CLOB
- Concatenable
- DATE
- DATETIME
- DECIMAL
- Date
- DateTime
- Enum
- FLOAT
- Float
- INT
- INTEGER
- Integer
- Interval
- LargeBinary
- MatchType
- NCHAR
- NVARCHAR
- Numeric
- PickleType
- REAL
- SMALLINT
- SchemaType
- SmallInteger
- String
- TEXT
- TIME
- TIMESTAMP
- Text
- Time
- TypeDecorator
- TypeEnginBases
- TypeEngine
- Unicode
- VARBINARY
- VARCHAR
---

### Other Types

Beside the default SQLAlchemy column types, ActiveAlchemy also extends the types with some additional ones for convenience from SQLAlchemy-Utils.

---

#### DateTime

Alias to ArrowType, which provides way of saving **[Arrow](https://arrow.readthedocs.io/en/latest/)** objects into database.
It automatically changes Arrow objects to datetime objects on the way in and
datetime objects back to Arrow objects on the way out (when querying database).

*Example*

```
# __models__.py

from assembly import db

class Article(db.Model):
    title = db.Column(db.String(255))
    published_at = db.Column(db.DateTime)

```

** Create **

```
from assembly import date, models

models.Article.create(title='Hi', published_at=date.utcnow)

```

** Read **

```
from assembly import models

article = models.Article.get(1)

article.created_at = article.created_at.replace(hours=-1)

article.created_at.humanize()
#-> 'an hour ago'
```

** Links **

<a href="https://sqlalchemy-utils.readthedocs.io/en/latest/data_types.html#module-sqlalchemy_utils.types.arrow" target="_blank">ArrowType</a>

<a href="https://arrow.readthedocs.io/en/latest/" target="_blank">Arrow</a>

<a href="https://arrow.readthedocs.io/en/latest/#tokens" target="_blank">Arrow Tokens</a>


---

#### StorageObjectType

StorageObjectType offers way of saving `Storage.Object` data structures to database.
It automatically changes Arrow objects to JSON objects on the way in and
`Storage.Object` objects back on the way out (when querying database).

*Example*

```
from assembly import db

class Article(db.Model):
    title = db.Column(db.String(255))
    published_at = db.Column(db.DateTime)
    image = db.Column(db.StorageObjectType)
```

** Create **

```
from assembly import date, models, asm

image_file = "file/xyz.jpg"
image = asm.upload_file(image_file)
models.Article.create(title='Hi', published_at=date.utcnow, image=image)
```

** Read **

```
from assembly import models

article = models.Article.get(1)

article.image.url
article.image.size
article.image.download()
```

---

#### EmailType

Provides a way for storing emails in a lower case.

*Example*

```
from assembly import db

class Article(db.Model):
    title = db.Column(db.String(255))
    published_at = db.Column(db.DateTime)
    email = db.Column(db.EmailType)
```

<a href="https://sqlalchemy-utils.readthedocs.io/en/latest/data_types.html#module-sqlalchemy_utils.types.email" target="_blank">EmailType</a>

---

#### JSONType

JSONType offers way of saving JSON data structures to database.
On PostgreSQL the underlying implementation of this data type is ‘json’
while on other databases its simply ‘text’.

*Example*

```
from assembly import db

class Article(db.Model):
    title = db.Column(db.String(255))
    published_at = db.Column(db.DateTime)
    data = db.Column(db.JSONType)
```

** Create **

```
from assembly import models, date

models.Article.create(title='Hello',
                        published_at=date.utcnow,
                        data={
                            "reference": "blah",
                            "tags": ["A", "B", "C"]
                        })
```

** Read **

```
article = models.Article.get(1)

article.data.get("reference")
# blah

article.data.get("tags")
# ["A", "B", "C"]

article.data.get("location", "Charlotte")
# Charlotte
```

<a href="https://sqlalchemy-utils.readthedocs.io/en/latest/data_types.html#module-sqlalchemy_utils.types.json" target="_blank">JSONType</a>

---

#### More SQLAlchemy-Utils Types

For more types, SQLAlchemy-Utils provides various new data types for SQLAlchemy.

<a href="https://sqlalchemy-utils.readthedocs.io/en/latest/data_types.html" target="_blank">SQLAlchemy-Utils</a>

- ArrowType
- ChoiceType
- ColorType
- CountryType
- CurrencyType
- EmailType
- EncryptedType
- JSONType
- LocaleType
- LtreeType
- IPAddressType
- PasswordType
- PhoneNumberType
- ScalarListType
- TimezoneType
- TSVectorType
- URLType
- UUIDType
- WeekDaysType


