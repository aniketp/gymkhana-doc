---
title: Local Development
tags: 
keywords:
summary: This is a Documentation for the Nominations Portal of Student Gymkhana, IIT-Kanpur. If you are in charge of maintaining the code or if you simply want the information about the application, this doc is for you.
permalink: mydoc_about_ruby_gems_etc.html
folder: mydoc
sidebar: mydoc_sidebar
---

## Clone the Repository
Clone the app-repository for local development

```
 git clone https://github.com/Gymkhana-Nominations.git
```

## Install dependencies
Navigate to the project folder and install the required tools to get started.

``` 
 cd Gymkhana-Nominations
```

```
 pip install -r requirements.txt
```

Here is the list of tools required. Content of **requirements.txt**
```
    django==1.11.2
    django-allauth==0.32.0
    django-bootstrap-admin==0.3.4
    django-bootstrap-form==3.2.1
```

## Run the app on localhost-server
```
 python manage.py runserver
```

The local version of app will be available at [localhost:8000](http://127.0.0.1:8000)
