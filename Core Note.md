Flask >> Decorator Based

Flask RESTful >> Class Based

### ```VENV``` Process

1. ```pip install virtualenv```
2. ```virtualenv venv --python=python3.8```
3. ```./venv/Scripts/activate.bat```
4. ```deactivate``

### Boilerplate Flask RESTful

```python
from flask import Flask
from flask_restful import Resource, Api

app =  flask(__name__)
api = api(app)

class Item(Resource):
  def get(self, name):
    return 200
  
  def post(self, name):
    data = request.get_json()
    return 201

api.add_resource(Student, '/student/<string:name>')
app.run(port=5000, debug=True)
```

### Lambda's for HTTP verbs

1. **GET** >> ```item = next(filter(lambda x: x['name'] == name, items), None)```
2. **POST** >> ```if next(filter(lambda x: x['name']==name, items), None) else {'name': name, 'price': data['price']} ```


### JWT

