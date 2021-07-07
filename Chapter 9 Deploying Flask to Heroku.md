# What is Heroku & Getting our code into Github

Dyno: Heroku's Virtual Machine. We can have more than one dyno on a service which makes the app run faster. But this means you have to additional dynos.

uWSGI: A python library that helps the Flask app to run.


Create a file called ```.gitignore``` in the root level. These contains "patterns" that is going to be ignore by git. The things we don't want to put in git will be added here. This can contain the following:

```
__pycache___/ #python cache memory
*.pyc #compiled python code
```

This file is constantly monitored by git. But when committing git doesn't add ```.gitignore``` automatically. You have to add it manually by ```git add .gitignore```. This is because files that start with ```.``` are recognized as system files and are ignored by git when committing.

# Restart from 6
