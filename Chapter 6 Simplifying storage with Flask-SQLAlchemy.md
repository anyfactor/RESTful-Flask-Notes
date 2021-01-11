### Welcome

**SQL Alchemy** is used to *easily* store objects to relational database.

### Setup

SQLAlchemy allows us to "Map our object to database rows". 

```bash
virtualenv venv --python=python3.5 # venv is the name of the virtual environment
./venv/Scripts/activate.bat
pip install flask
pip install Flask-JWT
pip install Flask-RESTful
pip install Flask-SQLAlchemy
```

### Improving the Project Structure and Maintainability

Create these folders -
- Models
- Resources

To convert these folders into *Python Packages* include a ```__init__.py``` script in to the directory.

```item.py``` and ```user.py``` both uses *Resource* and we should move these scripts to the "Resources" directory/package.

##### app.py


```python
from flask import Flask
from flask_restful import Api
from flask_jwt import JWT

from security import authenticate, identity
# from user import UserRegister
from resources.user import UserRegister
#from item import Item, ItemList
from resources.item import Item, ItemList

app = Flask(__name__)
app.secret_key = 'use a secret key here'
api = Api(app)

jwt = JWT(app, authenticate, identity)

api.add_resource(Item, '/item/<string:name>')
api.add_resource(ItemList, '/items')
api.add_resource(UserRegister, '/register') # Registering user signup in; /auth != /register

app.run(port=5000, debug=True)
```

##### security.py

```python

from werkzeug.security import safe_str_cmp
# from user import User
from resources.user import User

def authenticate(username, password):
  user = User.find_by_username(username)
  if user and safe_str_cmp(user.password, password)
    return user

def identity(payload):
  user_id = payload['identity']
  return User.find_by_id(user_id)
```

### Creating user and item model

Current file directory

```
models
- __init__.py
resources
- __init__.py
- item.py
- user.py
app.py
create_tables.py
security.py
```

In the resources directory/packages, the ```item.py``` file contains two classes. ```Item``` and ```ItemList```. And they are both "resources". In the same directory, the ```user.py``` file contains two classes. ```User``` and ```UserRegister```. Here only the ```UserRegister``` is the only resource class.

Now we need to remove the ```User``` class from the resource directory as it does not make logical sense to have it in the resource directory. We are going to cut that out and create a new file in the models folder called ```user.py``` and paste it there.

**What is a model is?** Resources are something that API responds with and API clients can ask for. Resources is everything the API deals with. The class user in models/user.py isn't something that the API can send or receive or can't convert to a JSON representation. This is a *helper*. A model our internal representation of the entity. A resource is the external representation of the entity. Models help us with extended flexibility with resources with making the resource file more complicated.

##### models/user.py

```python
import sqlite3

# class User:
class UserModel:
# Because model and resources are different and we need to make it different.
  def __init__(self, _id, username, password):
    self.id = _id
    self.username = username
    self.password = password

  @classmethod
  def find_by_username(cls, username):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "SELECT * FROM users WHERE username=?"
    result = cursor.execute(query, (username))
    row = result.fetchone()
    
    if row:
      user = cls(*row)
    else:
      user = None

    @classmethod
    def find_by_id(cls, _id):
      connection = sqlite3.connect('data.db')
      cursor = connection.cursor()

      query = "SELECT * FROM users WHERE id=?"
      result = cursor.execute(query, (_id))
      row = result.fetchone()

      if row:
        user = cls(*row)
      else:
        user = None

      connection.close()
      return user
```

##### resources/user.py

```python
import sqlite3
from flask_restful import Resource, reqparse
import models.user import UserModel
# We are importing this as we have created a new file

class UserRegister(Resource):
  parser = reqparse.RequestParser()
  parser.add_argument('username',
    type=str,
    required=True
    help="This field cannot be blank"
  )
  parser.add_argument('password',
    tyep=str,
    required=True,
    help="This field cannon be blank"
  )

  def post(self):
    data = UserRegister.parser.parse_args()

    # if User.find_by_username(data['username']):
    # Class name was changed
    if UserModel.find_by_username(data['username']):
      return {"message": "A user with that username already exists"}, 400
    
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO users VALUES (NULL, ?, ?)"
    cursor.execute(query, (data['username'], data['password']))

    connection.commit()
    connection.close()
    
    return {'message': 'User created successfully'}, 201
```


##### security.py

```python
from werkzeug.security import safe_str_cmp
# from resources.user import User
#  We created a separate file from this
from models.user import UserModel


def authenticate(username, password):
  user = UserModel.find_by_username(username) #this was changed
  if user and safe_str_cmp(userp.password, password):
    return user

def identity(payload):
  user_id =  payload['identity']
  return UserModel.find_by_id(user_id) #this was changed
```

Now we are going to separate the item part from resources to models. But we don't have an ```item``` mode so we are going to create one from scratch. Now, from the ```resources/item.py``` we have to figure out what things that can be done in the ```models/item.py``` to make that file less complicated. Resource should only deal with *request verbs* but additional functionalities through methods should be carried by the models file.

So we are taking out ```find_by_name_method```, ```insert``` method as they are not called by the API directly.  

##### models/item.py

```python
import sqlite3

class ItemModel:
# as it is an internal representation it should contain object properties
  def __init__(self, name, price):
    self.name = name
    self.price = price
  
  def json(self):
    return {'name': self.name, 'price': self.price}
    # dictionary representing our item

  # from resource/item to model/item
  @classmethod
  def find_by_name(cls, name):
   connection = sqlite3.connect('data.db')
   cursor = connection.cursor()

   query = "SELECT * FROM WHERE name=?"
   result = cursor.execute(query, (name))
   row = result.fetchone()
   connection.close()

   if row:
    return {'item': {'name': row[0], 'price': row[1]}}
  
  @classmethod
  def insert(cls, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO items VALUES (?, ?)"
    cursor.execute(query, (item['name'], item['price']))

    connection.commit()
    connection.close()

  @classmethod
  def update(cls, item):
  # Yes this is a model
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()
    
    query = "UPDATE items SET price=? WHERE name=?"
    cursor.execute(query, (item['price'], item['name']))

    connection.commit()
    connection.close()

```

Tip: Methods within a class should be separated by one line; classes by two lines.


##### resource/item.py

```python
from flask_restful import Resource, reqparse
from flask_jwt import jwt_required
import sqlite3
from models.item import ItemModel # Importing the new ItemModel Class

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
    item = ItemModel.find_by_name() # This line was changed from self > ItemModel
    if item:
      return item
    return {'message': 'item does not exist'}, 404  

  def post(self, name):
    # if self.find_by_name(name):
    if ItemModel.find_by_name(name): #This line was changed
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = {'name': name, 'price': data['price']}

    try:
      ItemModel.insert(item) # Changed
    except:
      return {'message': 'An error occurred'},
    return item, 201

  def delete(self, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "DELETE FROM items WHERE name=?"
    cursor.execute(query, (name,))
    connection.commit()
    connection.close()

  def put(self, name):
    data = Item.parser.parse_args()

    item = ItemModel.find_by_name(name) #changed
    updated_item = {'name': name, price: data['price']}

    if item is None:
      try:
        ItemModel.insert(updated_item) #changed
      except:
        return {"message": "An error ocurred inserting the item."}, 500
    else:
      try:  
        ItemModel.update(updated_item) #changed
      except:
        return {"message": "An error ocurred updating the item."}, 500
    return updated_item


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

Bear with me as this a bit a complicated.

The ```ItemModel``` IS the item that we are dealing with. Now, **CLS or classmethod is used when inside the class the class itself is being referenced.**

Now if we are dealing with the Item itself in *insert* or *update* the ```classmethod``` does not make sense. We are "initing" the class with objects of price and name in the ItemModel. These two models are inserting itself and a class method doesn't make any sense. 

We are going to change ```ItemModel.insert(updated_item)``` to simply ```updated_item.insert``` because the ItemModel was already inited with the item and it is not necessary to do a double with a ```classmethod```.


##### models/item.py

```python
import sqlite3

class ItemModel:
  def __init__(self, name, price):
    self.name = name
    self.price = price
  
  def json(self):
    return {'name': self.name, 'price': self.price}

  def find_by_name(cls, name):
  # As this returns an object rather than a dictionary we are going to need classmethod
  # We need to return object model instead a dictionary
   connection = sqlite3.connect('data.db')
   cursor = connection.cursor()

   query = "SELECT * FROM WHERE name=?"
   result = cursor.execute(query, (name))
   row = result.fetchone()
   connection.close()

   if row:
    # return {'item': {'name': row[0], 'price': row[1]}}
    # We need to return object model instead a dictionary
    return cls(*row)
    # This is essentially this return cls(row[0], row[1])
    # Argument unpacking
    # cls calls the ItemModels init method
    # row[0] = name. row[1] = price
  
  # @classmethod
  # def insert(cls, name):
  def insert(self):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO items VALUES (?, ?)"
    # cursor.execute(query, (item['name'], item['price']))
    cursor.execute(query, (self.name, self.price))

    connection.commit()
    connection.close()

  # @classmethod
  # def update(cls, item):
  def update(self):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()
    
    query = "UPDATE items SET price=? WHERE name=?"
    cursor.execute(query, (self.name, self.price))

    connection.commit()
    connection.close()

```

##### resource/item.py

```python
from flask_restful import Resource, reqparse
from flask_jwt import jwt_required
import sqlite3
from models.item import ItemModel

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
    item = ItemModel.find_by_name()
    # This now returns an Item object rather than a dictionary
    if item:
    # Now cannot return an item directly as this is not a dictionary/json
    # We have a method on ItemModel specifically for this called 'json'
      # return item
      return item.json()
    return {'message': 'item does not exist'}, 404  

  def post(self, name):
    if ItemModel.find_by_name(name):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    # item = {'name': name, 'price': data['price']}
    # Now it is an item model
    item = ItemModel(name, data['price'])

    try:
      # ItemModel.insert(item)
      # We can insert the item model
      item.insert()
    except:
      return {'message': 'An error occurred'},
    return item.json(), 201 #changed

  def delete(self, name):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "DELETE FROM items WHERE name=?"
    cursor.execute(query, (name,))
    connection.commit()
    connection.close()

  def put(self, name):
    data = Item.parser.parse_args()

    item = ItemModel.find_by_name(name)
    # This is an item object
    # updated_item = {'name': name, price: data['price']}
    # ^ This is an item dictionary
    # We convert this to item model
    updated_item = ItemModel(name, data['price'])

    if item is None:
      try:
        # ItemModel.insert(updated_item)
        updated_item.insert()
      except:
        return {"message": "An error ocurred inserting the item."}, 500
    else:
      try:  
        # ItemModel.update(updated_item)
        updated_item.update()
      except:
        return {"message": "An error ocurred updating the item."}, 500
    # return updated_item
    return updated_item.json()


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

### Advanced Postman Environments

**Setting Variables**

```
{{url}}/items
```
Gear logo > Manage Environments > Add Environment

Key | Value
---|---
url | 127.0.0.1/5000


Then remember to save.

**Handling the JWT token**

Headers > Authorization > ```JWT {{jwt_token}}```

In the endpoint select the "Tests" tab.

```javascript
var jsonData = JSON.parse(responseBody);
``` 

You can get more of these codes in the snippets section. Add -

```javascript
tests['access_token_test'] = jsonData.access_token !=== undefined;
```

Returns a Boolean value for checking if the response code exists or not. Add -

```javascript
postman.setEnvironmentVariable("jwt_token", jsonData.access_token)
```

Saves JWT token as environmental variable.

Send the request and you will see that your test passed and you even which ones passed.

If you click the "eye" logo on top you can see your environmental variables there.

Explore the snippet column on the tests section.

### Telling SQLAlchemy about our tables

Create a newfile in the basecode folder called ```db.py```.

##### db.py

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchamey()
# This object is going to link to the app.py
# then look into all the onjects we tell it to
# them map those objects in to rows of a database
```

##### app.py

```python
from flask import Flask
from flask_restful import Api
from flask_jwt import JWT

from security import authenticate, identity
from resources.user import UserRegister
from resources.item import Item, ItemList

app = Flask(__name__)
app.secret_key = "secret_key"
api = Api(app)

jwt = JWT(app, authenticate, identity) #/auth
api.add_resource(Item, '/item/<string:name>')
api.add_resource(ItemList, '/items')
api.add_resource(UserRegister, '/register')

if __name__ = "__main__":
  from db import db #Changed
  # circular imports
  # We are going to import the db in other modules
  # so this will import db once more if we put it in the top
  db.init_app(app)
  app.run(port=5000, debug=True)

```

##### models/user.py

```python
import sqlite3
from db import db

class UserModel(db.Model): #changed
  # This db.model tell SQLAlchemy that we will be
  # interacting with this class/ item models/ objects in the database
  # This creates a 'mapping' between objects and the database

  __tablename__ = 'users'

  # This 'users' is the table row we are accessing in the db
  # The columns we will be accessing >

  id = db.Column(db.Integer, primary_key=True)
  # primary_key> unique indexable variable
  username = db.Column(db.String(80))
  # Limits the username string size
  password = db.Column(db.String(80))
  # This variables must be identical to the init variable
  # Anything that is not declared here won't be communicated with the db

  def __init__(self, _id, username, password):
    self.id = _id
    self.username = username
    self.password = password

  @classmethod
  def find_by_username(cls, username):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "SELECT * FROM users WHERE username=?"
    result = cursor.execute(query, (username))
    row = result.fetchone()
    
    if row:
      user = cls(*row)
    else:
      user = None
    
    connection.close()
    return user

    @classmethod
    def find_by_id(cls, _id):
      connection = sqlite3.connect('data.db')
      cursor = connection.cursor()

      query = "SELECT * FROM users WHERE id=?"
      result = cursor.execute(query, (_id))
      row = result.fetchone()

      if row:
        user = cls(*row)
      else:
        user = None

      connection.close()
      return user
```

##### models/item.py

```python
import sqlite3
from db import db #changed

class ItemModel(db.Model): #changed
  __tablename__ = 'items'

  id = db.Column(db.Integer, primary_key=True)
  # Even though we didn't have an id, we are going to create one anyway
  # it is very useful
  name = db.Column(db.String(80))
  price = db.Column(db.Float(precision=2))

  def __init__(self, name, price):
    self.name = name
    self.price = price
  
  def json(self):
    return {'name': self.name, 'price': self.price}

  def find_by_name(cls, name):
   connection = sqlite3.connect('data.db')
   cursor = connection.cursor()

   query = "SELECT * FROM items WHERE name=?"
   result = cursor.execute(query, (name))
   row = result.fetchone()
   connection.close()

   if row:
    return cls(*row)
  
  def insert(self):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()

    query = "INSERT INTO items VALUES (?, ?)"
    cursor.execute(query, (self.name, self.price))

    connection.commit()
    connection.close()

  def update(self):
    connection = sqlite3.connect('data.db')
    cursor = connection.cursor()
    
    query = "UPDATE items SET price=? WHERE name=?"
    cursor.execute(query, (self.name, self.price))

    connection.commit()
    connection.close()

```

##### app.py

```python
from flask import Flask
from flask_restful import Api
from flask_jwt import JWT

from security import authenticate, identity
from resources.user import UserRegister
from resources.item import Item, ItemList

app = Flask(__name__)

app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
# We are just disabling the resource intensive tracking of changes to objects
# This is a Flask tracker not the SQLAlchemy Tracker

app.secret_key = "secret_key"
api = Api(app)

jwt = JWT(app, authenticate, identity) #/auth
api.add_resource(Item, '/item/<string:name>')
api.add_resource(ItemList, '/items')
api.add_resource(UserRegister, '/register')

if __name__ = "__main__":
  from db import db
  db.init_app(app)
  app.run(port=5000, debug=True)

```

### Implementing ItemModel

##### create_tables.py

```python
import sqlite3

connection = sqlite3.connect('data.db')
cursor = connection.cursor()

create_table = "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, username text, password text)"
cursor.execute(create_table)

# create_table = "CREATE TABLE IF NOT EXISTS items (name text, price text)"
create_table = "CREATE TABLE IF NOT EXISTS items (id INTEGER PRIMARY KEY, name text, price text)"
cursor.execute(create_table)

connection.commit()
connection.close()
```

We need to run this file and create the db first. We need to have the db in the root directory.

##### models/item.py

```python
# import sqlite3
# Not needed anymore
from db import db

class ItemModel(db.Model):
  __tablename__ = 'items'

  id = db.Column(db.Integer, primary_key=True)
  name = db.Column(db.String(80))
  price = db.Column(db.Float(precision=2))

  def __init__(self, name, price):
    self.name = name
    self.price = price
  
  def json(self):
    return {'name': self.name, 'price': self.price}

  @classmethod
  def find_by_name(cls, name):
  # SQLAlchemy automatically converts the SQL row to an object

  #  connection = sqlite3.connect('data.db')
  #  cursor = connection.cursor()

  #  query = "SELECT * FROM items WHERE name=?"
  #  result = cursor.execute(query, (name))
  #  row = result.fetchone()
  #  connection.close()

  #  if row:
  #   return cls(*row)
  return cls.query.filter_by(name=name).first() # SELECT * FROM items WHERE name=name LIMIT 1
  # This '.query' comes from db.model that the ItemModel is taking as an argument.
  # This is an 'query builder' and we can build queries with this
  # 'filter_by' is query builder also which returns objects but we only need the first one
  # filter_by(name=name) # SELECT * FROM items WHERE name=name
  # We can chain `filter_by' functions together like-
  # filter_by(name=name).filter_by(id=1)
  # Or it can take multiple arguments rather than chaining
  # filter_by(name=name, id=1)
  # This is returning an ItemModel object
  # that means this is "inited" by the classmodel and it has self.name and self.price
  # return ItemModel.something is the same as cls.something but the thing it returns also gets passed through init


  
  # def insert(self):
  def save_to_db(self):
  # This is just saving the ItemModel to the database
  # SQLAlchemy can directly translate class object to database row
  # We don't need to define what row data to insert (self.name, self.price)
  # Just directly insert them into the db
  # The object we are dealing with is "self"

    # connection = sqlite3.connect('data.db')
    # cursor = connection.cursor()

    # query = "INSERT INTO items VALUES (?, ?)"
    # cursor.execute(query, (self.name, self.price))

    # connection.commit()
    # connection.close()
    db.session.add(self)
    db.session.commit()
    # session is a collection object we are going to write to the database
    # data > ItemModel > self/db.Model > init > (self.price, self.name) > session.add(self) > session.commit()
    # because of the unique primary ID this code can also update
    # that is why we changed the name
    # update + insert = upserting

  # Upserting makes this section redundant
  # def update(self):
  #   connection = sqlite3.connect('data.db')
  #   cursor = connection.cursor()
    
  #   query = "UPDATE items SET price=? WHERE name=?"
  #   cursor.execute(query, (self.name, self.price))

  #   connection.commit()
  #   connection.close()

  # New Stuff
  def delete_from_db(self):
    db.session.delete(self)
    db.session.commit()

```

##### models/item.py

```python
from db import db

class ItemModel(db.Model):
  __tablename__ = 'items'

  id = db.Column(db.Integer, primary_key=True)
  name = db.Column(db.String(80))
  price = db.Column(db.Float(precision=2))

  def __init__(self, name, price):
    self.name = name
    self.price = price
  
  def json(self):
    return {'name': self.name, 'price': self.price}

  @classmethod
  def find_by_name(cls, name):
    return cls.query.filter_by(name=name).first() # SELECT * FROM items WHERE name=name LIMIT 1

  def save_to_db(self):
    db.session.add(self)
    db.session.commit()

  def delete_from_db(self):
    db.session.delete(self)
    db.session.commit()

```

##### resource/item.py

```python
from flask_restful import Resource, reqparse
from flask_jwt import jwt_required
import sqlite3
from models.item import ItemModel

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
    item = ItemModel.find_by_name()
    if item:
      return item.json()
    return {'message': 'item does not exist'}, 404  

  def post(self, name):
    if ItemModel.find_by_name(name):
      return {'message': "An item with name '{}' already exists.".format(name)}, 400
    
    data = Item.parser.parse_args()
    item = ItemModel(name, data['price'])

    try:
      # item.insert()
      item.save_to_db()
    except:
      return {'message': 'An error occurred'},
    return item.json(), 201 

  def delete(self, name):
  # We are going to use the model to do the deletion
    # connection = sqlite3.connect('data.db')
    # cursor = connection.cursor()

    # query = "DELETE FROM items WHERE name=?"
    # cursor.execute(query, (name,))
    # connection.commit()
    # connection.close()
  item = ItemModel.find_by_name(name)
  if item:
    item.delete_from_db()
  return {"message": "Item Deleted"}


  def put(self, name):
    data = Item.parser.parse_args()

    item = ItemModel.find_by_name(name)
    # updated_item = ItemModel(name, data['price'])

    if item is None:
      # try:
      #   updated_item.insert()
      # except:
      #   return {"message": "An error ocurred inserting the item."}, 500
      item = ItemModel(name, data['price'])
      # we are creating a new item through the ItemModel class > db.model
    else:
      # try:  
      #   updated_item.update()
      # except:
      #   return {"message": "An error ocurred updating the item."}, 500
      item.price = data['price']
    
    item.save_to_db()
    # Because the item is now uniquely identified by db
    # all we have to do is save to db
    # return updated_item.json()
    return item.json()


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

Now we are going to tell flask where to find the ```data.db``` file.

##### app.py

```python
from flask import Flask
from flask_restful import Api
from flask_jwt import JWT

from security import authenticate, identity
from resources.user import UserRegister
from resources.item import Item, ItemList

app = Flask(__name__)

app.config['SQLALCHEMY_DATABSE_URI'] = 'sqlite:///data.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

app.secret_key = "secret_key"
api = Api(app)

jwt = JWT(app, authenticate, identity) #/auth
api.add_resource(Item, '/item/<string:name>')
api.add_resource(ItemList, '/items')
api.add_resource(UserRegister, '/register')

if __name__ = "__main__":
  from db import db
  db.init_app(app)
  app.run(port=5000, debug=True)

```

### Implementing the UserModel

```python
import sqlite3
from db import db

class UserModel(db.Model):
  __tablename__ = 'users'

  id = db.Column(db.Integer, primary_key=True)
  username = db.Column(db.String(80))
  password = db.Column(db.String(80))

  # def __init__(self, _id, username, password):
  def __init__(self, username, password):
    # self.id = _id
    # This id is automatically assigned
    # if we didn't autoincrement we could have used an id
    # We can also UUID
    self.username = username
    self.password = password

  @classmethod
  def find_by_username(cls, username):
    # connection = sqlite3.connect('data.db')
    # cursor = connection.cursor()

    # query = "SELECT * FROM users WHERE username=?"
    # result = cursor.execute(query, (username))
    # row = result.fetchone()
    
    # if row:
    #   user = cls(*row)
    # else:
    #   user = None

    # connection.close()
    # return user
    return cls.query.filter_by(username=username).first()

    @classmethod
    def find_by_id(cls, _id):
      # connection = sqlite3.connect('data.db')
      # cursor = connection.cursor()

      # query = "SELECT * FROM users WHERE id=?"
      # result = cursor.execute(query, (_id))
      # row = result.fetchone()

      # if row:
      #   user = cls(*row)
      # else:
      #   user = None

      # connection.close()
      # return user
      return cls.query.filter_by(id=_id).first()

    def save_to_db(self):
      db.session.add(self)
      db.session.commit()
    
    def delete_from_db(self):
      db.session.delete(self)
      db.session.commit()
```

##### resources/user.py

```python
import sqlite3
from flask_restful import Resource, reqparse
import models.user import UserModel

class UserRegister(Resource):
  parser = reqparse.RequestParser()
  parser.add_argument('username',
    type=str,
    required=True
    help="This field cannot be blank"
  )
  parser.add_argument('password',
    tyep=str,
    required=True,
    help="This field cannon be blank"
  )

  def post(self):
    data = UserRegister.parser.parse_args()

    if UserModel.find_by_username(data['username']):
      return {"message": "A user with that username already exists"}, 400
    
    # connection = sqlite3.connect('data.db')
    # cursor = connection.cursor()

    # query = "INSERT INTO users VALUES (NULL, ?, ?)"
    # cursor.execute(query, (data['username'], data['password']))

    # connection.commit()
    # connection.close()
    
    user = UserModel(**data)
    # This is exactly this
    # user = UserModel(data['username'], data['password'])
    # ** unpacks the values
    user.save_to_db()
    
    return {'message': 'User created successfully'}, 201
```

# Start from 11