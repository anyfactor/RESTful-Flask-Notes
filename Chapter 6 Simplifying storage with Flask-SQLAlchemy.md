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