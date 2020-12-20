### Runnings a SQLite database

```python
import sqlite3

connection = sqlite3.connect('data.db')
# connection establishes the connection between the module and the db

cursor = connection.cursor()
# cursor is like GUI cursor. It allows to select, retrieve, store and manipulate things (db).

create_table = "CREATE TABLE users(id int, username text, password text)"
# CREATE TABLE is the command
# users is the name of the table
# id, username, password is the columns

cursor.execute(create_table)
# we pass the command (variable) to the cursor > connection > data.db
# Now the db has been created

user = (1, 'abd', 'asdf')
insert_query = "INSERT INTO users VALUES (?, ?, ?)"
# SQL query (command)
# INSERT INTO > command
# users > location/table
# (?, ?, ?) > place things there. Think of it as "{} {} {}".format(*stuff)

cursor.execute(insert_query, user)


users = [
  (2, 'bad', 'asdf'),
  (3, 'cad', 'gasfdg')
]

cursor.executemany(insert_query, users)
# iterates and inserts all of them

select_query = "SELECT * FROM users"
# SELECT > Select for retrieving data
# * > Column name. "*" means all; you can name specific column too like "id"
# users > table name

for row in cursor.execute(select_query):
# we execute the command and it returns a list of rows(tuples) from the db
  print(row)

connection.commit() # more like connection.save()
connection.close() # need to close it
```


### Logging and retrieving Users from Database

We are going to use the db we created previously.

##### user.py

```python
import sqlite3

class User:
  def __init__(self, _id, username, password):
    self.id = _id
    self.username =  username
    self.password =  password
  
  def find_by_username(self, username):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "SELECT * from users WHERE username=?"
    # WHERE > limits rows where username is equal to something/parameter

    result = cursor.execute(query, (username,))
    # we need always pass params to SQL in tuple
    # (username,) > just having ('bob') == 'bob'; adding the trailing comma converts it to a tuple ('bob')

    row = result.fetchone() # Gets the first result from the results iterable (a bit like next); but it returns None automatically when empty
    if row: # if row is not None
      user = User(row[0], row[1], row[2])
      # adds to the User class object
      # _id, username, password = (2, 'bad', 'asdf')
    else:
      user = None

    connection.close() # don't need to commit as we did not write
    return user # User object or None
```


### CLS implication in logging and retrieving

CLS or classmethod is used when inside the class the class itself is being referenced.

##### user.py

```python
import sqlite3

class User:
  def __init__(self, _id, username, password):
    self.id = _id
    self.username =  username
    self.password =  password
  
  @classmethod # new add
  def find_by_username(cls, username): # self changed to cls
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()
    query = "SELECT * from users WHERE username=?"
    result = cursor.execute(query, (username,))

    row = result.fetchone()
    if row:
      user = cls(*row)
      # converted User class to cls
      # converted the individual tuple indexing to tuple spread using *row
    else:
      user = None

    connection.close()
    return user

    @classmethod
    def find_by_id(cls, _id):
      connection = sqlite3.connect('data.db')
      cursor = connection.cursor()
      query = "SELECT * from users WHERE id=?"
      result = cursor.execute(query, (_id,))

      row = result.fetchone()
      if row:
        user = cls(*row)
      else:
        user = None

      connection.close()
      return user
```

##### security.py

```python

from werkzeug.security import safe_str_cmp
from user import User


## This section is transferred to db
# users = [
#   User(1, 'bob', 'asdf')
# ]

## This section is removed by User.find_by_username & User.find_by_id
# username_mapping = {u.username: u for u in users}
# userid_mapping = {u.id: u for u in users}


def authenticate(username, password):
  user = User.find_by_username(username) # From User class
  if user and safe_str_cmp(user.password, password)
    return user

def identity(payload):
  user_id = payload['identity']
  return User.find_by_id(user_id) # From User class
```

### Signing up and writing users to database

##### create_tables.py

```python
import sqlite3

connection = sqlite3.connect('data.db')
cursor = connection.cursor()

# create_table = "CREATE TABLE IF NOT EXISTS users (id int, username text, password text)"
create_table = "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, username text, password text)"
# CREATE TABLE // IF NOT EXISTS are two different commands
# We don't want to "manually" add ids; we should auto increment this column
# INTEGER allows 'auto incrementing column'; int does not
# Now we just need to assign username and password; id will be auto incremented.

cursor.execute(create_table)

connection.commit()
connection.close()
```

##### users.py

We need to allow users to sign up. The existing ```User``` class allows to find users.

But we are going to create a separate system that allows the users to sign up. We are going to use **flask resource** for that.

```python
import sqlite3
import flask_restful import Resource, reqparse

class User:
  def __init__(self, _id, username, password):
    self.id = _id
    self.username =  username
    self.password =  password
  
  @classmethod
  def find_by_username(cls, username):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()
    query = "SELECT * from users WHERE username=?"
    result = cursor.execute(query, (username,))
    row = result.fetchone()
    user = cls(*row) if row else None
    connection.close()
    return user

    @classmethod
    def find_by_id(cls, _id):
      connection = sqlite3.connect('data.db')
      cursor = connection.cursor()
      query = "SELECT * from users WHERE id=?"
      result = cursor.execute(query, (_id,))
      row = result.fetchone()
      user = cls(*row) if row else None
      connection.close()
      return user

class UserRegister(Resource): #/register
# The user signup resource class
  parser = reqparse.RequestParser()
  parser.add_argument(
    'username',
    type=str,
    required=True,
    help="This field is required"
  )
  parser.add_argument(
    'password',
    type=str,
    required=True,
    help="This field is required"
  )
  def post(self):
    data = UserRegister.parser.add_args()

    if User.find_by_username(data['username']):
      return {'message': "A user with that username already exists"}, 400
    # if the user does not exist we create

    connection =  sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO users VALUES (NULL, ?, ?)"
    # INSERT INTO the users table
    # VALUES > command
    # We have put NULL there for the auto incrementing id. We can't even leave it empty.
    # The question marks are where we are going to put in data using tuples

    cursor.execute(query, (data['username'], data['password']))
    
    connection.commit()
    connection.close()

    return {'message': 'User created successfully'}
```

##### app.py


```python
from flask import Flask, request
from flask_restful import Resource, Api
from flask_jwt import JWT, jwt_required
from flask_restful import reqparse

from security import authenticate, identity
from user import UserRegister

app = Flask(__name__)
app.secret_key = 'use a secret key here'
api = Api(app)
jwt = JWT(app, authenticate, identity)

items = []
Class Item(Resource):
  parser =  reqparse.RequestParser()
  parser.add_argument(
      'price',
      type=float,
      required=True,
      help="this field cannot be left empty"
  )

  @jwt_required()
  def get(self, name):
    item = next(filter(lambda x: x['name'] == name, items), None)
    return {'item': item}, 200 if item else 404
  
  def post(self, name):
    if next(filter(lambda x: x['name']==name, items), None):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = {'name': name, 'price': data['price']}
    items.append(item)
    return item, 201

  def delete(self, name):
     global items
     items = list(filter(lambda x: x['name']==name, items))
     return {'message': 'item deleted'}
  
  def put(self, name):
    item = next(filter(lambda x: x['name'] == name, items), None)
    data = parser.parse_args()

    if item is None:
      item = {'name': name, price: data['price']}
      items.append(item)
    else:
      item.update(data)
    return item

class ItemList(Resource):
  def get(self):
    return {'items': items}
  

api.add_resource(Item, '/item/<string:name>')
api.add_resource(ItemList, '/items')
api.add_resource(UserRegister, '/register') # Registering user signup in; /auth != /register

app.run(port=5000, debug=True)
```

##### Postman

We will POST to "/register" to create the user. Then we will POST to "/auth" to get the "access token" then we can use it.

### Retriving our item resources from a database

Cut and paste the ```item``` and ```itemlist``` resources from ```app.py``` and paste them into a new file ```item.py```

##### item.py

```python
from flask_restful import Resoruce, reqparse
from flask_jwt import jwt_required
import sqlite3

Class Item(Resource):
  parser =  reqparse.RequestParser()
  parser.add_argument(
      'price',
      type=float,
      required=True,
      help="this field cannot be left empty"
  )

  @jwt_required()
  def get(self, name):
    # item = next(filter(lambda x: x['name'] == name, items), None)
    # return {'item': item}, 200 if item else 404
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor

    query = "SELECT * FROM items WHERE name=?"
    result = cursor.execute(query, (name,))
    row = result.fetchone()
    connection.close()

    if row:
      return {'item': {'name': row[0], 'price':row[1]}}
    else:
      return {'message': 'item does not exist'}, 404  
  
  def post(self, name):
    if next(filter(lambda x: x['name']==name, items), None):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = {'name': name, 'price': data['price']}
    items.append(item)
    return item, 201

  def delete(self, name):
     global items
     items = list(filter(lambda x: x['name']==name, items))
     return {'message': 'item deleted'}
  
  def put(self, name):
    item = next(filter(lambda x: x['name'] == name, items), None)
    data = parser.parse_args()

    if item is None:
      item = {'name': name, price: data['price']}
      items.append(item)
    else:
      item.update(data)
    return item

class ItemList(Resource):
  def get(self):
    return {'items': items}
```

##### app.py

```python
# from flask import Flask, request  # Don't need request
from flask import Flask
# from flask_restful import Resource, Api, reqparse # Don't need resource, reqpares
from flask_restful import Api
# from flask_jwt import JWT, jwt_required # Don't need jwt_required
from flask_jwt import JWT

from security import authenticate, identity
from user import UserRegister
from item import Item, Itemlist # import the classes (resources) from item.py

app = Flask(__name__)
app.secret_key = 'use a secret key here'
api = Api(app)
jwt = JWT(app, authenticate, identity)

#  items = [] # Don't need it
  

api.add_resource(Item, '/item/<string:name>')
api.add_resource(ItemList, '/items')
api.add_resource(UserRegister, '/register') # Registering user signup in; /auth != /register

if __name__ = '__main___':
  app.run(port=5000, debug=True)
# Why?
# In python when you import something python runs that file
# but except for the bit in this section here.
# You have to be in this file first to run it.
```

##### create_tables.py

```python
import sqlite3

connection = sqlite3.connect('data.db')
cursor = connection.cursor()

create_table = "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, username text, password text)"
cursor.execute(create_table)

create_table = "CREATE TABLE IF NOT EXISTS items (name text, price real)" # real is real number
cursor.execute(create_table)

cursor.execute("INSERT INTO items values ('test', 10.99)")

connection.commit()
connection.close()
```

### Writing an item

Because we check the existence of an item first in both POST and GET it is better to have it as its own function

##### item.py

```python
from flask_restful import Resoruce, reqparse
from flask_jwt import jwt_required
import sqlite3

Class Item(Resource):
  parser =  reqparse.RequestParser()
  parser.add_argument(
      'price',
      type=float,
      required=True,
      help="this field cannot be left empty"
  )

  @jwt_required()
  def get(self, name):
    item = self.find_by_name() # You can call it a Item.find_by_name but we called as classmethod so we can do self.find_by_name
    if item:
      return item
    return {'message': 'item does not exist'}, 404  
  
  @classmethod
  def find_by_name(cls, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor

    query = "SELECT * FROM items WHERE name=?"
    result = cursor.execute(query, (name,))
    row = result.fetchone()
    connection.close()

    if row:
      return {'item': {'name': row[0], 'price':row[1]}}

  def post(self, name):
    if self.find_by_name(name):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = {'name': name, 'price': data['price']}

    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO items VALUES (?, ?)"
    cursor.execute(query, (item['name'], item['price']))
    connection.commit()
    connection.close()

    return item, 201

  def delete(self, name):
    #  global items
    #  items = list(filter(lambda x: x['name']==name, items))
    #  return {'message': 'item deleted'}
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "DELETE FROM items WHERE name=?" # Important to WHERE then specific element
    cursor.execute(query, (name,))
    connection.commit()
    connection.close()

  def put(self, name):
    item = next(filter(lambda x: x['name'] == name, items), None)
    data = parser.parse_args()

    if item is None:
      item = {'name': name, price: data['price']}
      items.append(item)
    else:
      item.update(data)
    return item

class ItemList(Resource):
  def get(self):
    return {'items': items}
```

### Refactoring insertion of items


##### item.py

```python
from flask_restful import Resoruce, reqparse
from flask_jwt import jwt_required
import sqlite3

Class Item(Resource):
  parser =  reqparse.RequestParser()
  parser.add_argument(
      'price',
      type=float,
      required=True,
      help="this field cannot be left empty"
  )

  @jwt_required()
  def get(self, name):
    item = self.find_by_name()
    if item:
      return item
    return {'message': 'item does not exist'}, 404  
  
  @classmethod
  def find_by_name(cls, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor

    query = "SELECT * FROM items WHERE name=?"
    result = cursor.execute(query, (name,))
    row = result.fetchone()
    connection.close()

    if row:
      return {'item': {'name': row[0], 'price':row[1]}}

  def post(self, name):
    if self.find_by_name(name):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = {'name': name, 'price': data['price']}

#  We will remove this to refactor the code
    # connection = sqlite3.connect('data.db')
    # cursor = connection.cursor()

    # query = "INSERT INTO items VALUES (?, ?)"
    # cursor.execute(query, (item['name'], item['price']))
    # connection.commit()
    # connection.close()

    # there is a possibility of error so better try and except
    try:
      self.insert(item)
    except:
      return {'message': 'An error occurred'}, 500 # Internal sever error
    return item, 201

  @classmethod
  def insert(cls, item):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO items VALUES (?, ?)"
    cursor.execute(query, (item['name'], item['price']))
    connection.commit()
    connection.close()

  def delete(self, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "DELETE FROM items WHERE name=?"
    cursor.execute(query, (name,))
    connection.commit()
    connection.close()

  def put(self, name):
    data = Item.parser.parse_args()
    item = self.find_by_name(name)

    updated_item = {'name': name, price: data['price']}

    if item is None:
      item = {'name': name, price: data['price']}
      items.append(item)
    else:
      item.update(data)
    return item

class ItemList(Resource):
  def get(self):
    return {'items': items}
```

### The PUT Method & Retrieving Items


##### item.py

```python
from flask_restful import Resoruce, reqparse
from flask_jwt import jwt_required
import sqlite3

Class Item(Resource):
  parser =  reqparse.RequestParser()
  parser.add_argument(
      'price',
      type=float,
      required=True,
      help="this field cannot be left empty"
  )

  @jwt_required()
  def get(self, name):
    item = self.find_by_name()
    if item:
      return item
    return {'message': 'item does not exist'}, 404  
  
  @classmethod
  def find_by_name(cls, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor

    query = "SELECT * FROM items WHERE name=?"
    result = cursor.execute(query, (name,))
    row = result.fetchone()
    connection.close()

    if row:
      return {'item': {'name': row[0], 'price':row[1]}}

  def post(self, name):
    if self.find_by_name(name):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = {'name': name, 'price': data['price']}

    try:
      self.insert(item)
    except:
      return {'message': 'An error occurred'},
    return item, 201

  @classmethod
  def insert(cls, item):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO items VALUES (?, ?)"
    cursor.execute(query, (item['name'], item['price']))
    connection.commit()
    connection.close()

  def delete(self, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "DELETE FROM items WHERE name=?"
    cursor.execute(query, (name,))
    connection.commit()
    connection.close()

  def put(self, name):
    data = Item.parser.parse_args()

    item = self.find_by_name(name)
    updated_item = {'name': name, price: data['price']}

    if item is None:
      try:
        self.insert(updated_item)
      except:
        return {"message": "An error ocurred inserting the item."}, 500
    else:
      try:  
        self.update(updated_item)
      except:
        return {"message": "An error ocurred updating the item."}, 500
    return updated_item

  @classmethod
  def update(cls, item):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "UPDATE items SET price=? WHERE name=?"
    cursor.execute(query, (item['price'], item['name']))

    connection.commit()
    connection.close()
```


### Retrieving Items


##### item.py

```python
from flask_restful import Resoruce, reqparse
from flask_jwt import jwt_required
import sqlite3

Class Item(Resource):
  parser =  reqparse.RequestParser()
  parser.add_argument(
      'price',
      type=float,
      required=True,
      help="this field cannot be left empty"
  )

  @jwt_required()
  def get(self, name):
    item = self.find_by_name()
    if item:
      return item
    return {'message': 'item does not exist'}, 404  
  
  @classmethod
  def find_by_name(cls, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor

    query = "SELECT * FROM items WHERE name=?"
    result = cursor.execute(query, (name,))
    row = result.fetchone()
    connection.close()

    if row:
      return {'item': {'name': row[0], 'price':row[1]}}

  def post(self, name):
    if self.find_by_name(name):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = {'name': name, 'price': data['price']}

    try:
      self.insert(item)
    except:
      return {'message': 'An error occurred'},
    return item, 201

  @classmethod
  def insert(cls, item):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO items VALUES (?, ?)"
    cursor.execute(query, (item['name'], item['price']))
    connection.commit()
    connection.close()

  def delete(self, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "DELETE FROM items WHERE name=?"
    cursor.execute(query, (name,))
    connection.commit()
    connection.close()

  def put(self, name):
    data = Item.parser.parse_args()

    item = self.find_by_name(name)
    updated_item = {'name': name, price: data['price']}

    if item is None:
      try:
        self.insert(updated_item)
      except:
        return {"message": "An error ocurred inserting the item."}, 500
    else:
      try:  
        self.update(updated_item)
      except:
        return {"message": "An error ocurred updating the item."}, 500
    return updated_item

  @classmethod
  def update(cls, item):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "UPDATE items SET price=? WHERE name=?"
    cursor.execute(query, (item['price'], item['name']))

    connection.commit()
    connection.close()


class ItemList(Resource):
  def get(self):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "SELECT * FROM items"
    result = cursor.execute(query)
    items = []
    for row in result:
      items.append(
        {'name': row[0], 'price': row[1]}
      )

    connection.close()

    return {'items': items}
```