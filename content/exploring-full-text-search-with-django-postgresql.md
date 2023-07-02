Title: Exploring Full-Text Search with Django/PostgreSQL
Date: 2023-06-21
Category: Development
Slug: exploring-full-text-search-with-django-postgresql
Tags: python, django, sql
Banner: https://i.imgur.com/r6XewFw.png


In this step-by-step tutorial, you'll learn how to use full-text search in your Django application using the PostgreSQL database. By the end of this article, you will have a thorough understanding of the use of full-text search in your application. This article requires to know the ORM and Queryset functions of Django well.

# What is Full-Text Search?

Before I answer that, let me ask you something. What is the difference between the find feature in a word processor (MS Word, Notepad) and search in Google? Both are searching, but the difference is how they are done. Google takes your query breaks it down into different words and crawls every website to get the relevant results, on the other hand, Find-and-Replace takes your query as a whole and searches for that chunk in a text. The former is a Full-Text search. It breaks down the query and removes unnecessary parts such as in, or, and then returns the most relevant results.

## Why Full-Text Search?

Let's understand the need of a full-text search using a real-life example. Suppose we have two posts with the given title and the user looks for «async in Python».

### Sample Post title in out app.

```plaintext
- Python also supports async function.
- Async in python is amazing
```

The user will only get the second post when searching in normal regular expression search, but we can see that first article is also related to the user's query. That's the reason we need full-text search in our applications.

# Features of Full-Text Search

* It is much faster.
    
* Results are relevant.
    
* Better ranking of results.
    

## Searching in Python

Before delving into Full-Text Search, I want to quickly review the normal workings of the SQLite database search.

Below you will find the repository to obtain the Boilerplate. I created a new project with django called core where I created a new app called a blog. After running `python manage.py migrate`, add this code into your project.

**blog/models.py** A new model (table) to store content and title of a blog post.

```plaintext
from django.db import models

# Create your models here.
class Post(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()

    def __str__(self):
        return f"{self.title}"
```

**blog/admin.py** Add Post model in admin site to use Create/Read/Update/Delete operations.

```plaintext
from django.contrib import admin
from .models import Post

# Register your models here.

admin.site.register(Post)
```

**blog.views.py** Add home view to see all the posts

```plaintext
from django.shortcuts import render
from .models import Post

def home(request):
    qs = Post.objects.all()
    if request.GET.get('query'):
        qs = qs.filter(title__icontains=query)
    return render(request, "index.html", context={"queryset": qs})
```

**core/urls.py** Add a new URL for home view.

```plaintext
urlpatterns = [
    path("admin/", admin.site.urls),
    path("", views.home),
]
```

### Template to search

blog/templates/index.html A template to search and see all the post in the queryset

```plaintext
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Full Text Search</title>
  </head>
  <body>
      <form>
           {% csrf_token %}
          <input type="text" name="query">
          <input type="submit" value="Search">
    <form/>
    {% for object in queryset %}
    <h2>{{object.title}}</h2>
    <p>{{object.content | truncatechars:200}}</p>
    {% empty %}
    <h2>No Results</h2>
    {% endfor %}
  </body>
</html>
```

After adding these changes to your project, launch this command in prompt/terminal.

```plaintext
python manage.py runserver
```

Goto http://127.0.0.1:8000/admin/blog/post/ and add any five posts on any topics like this.

![Admin Screen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/40g94luqkp0lz37s9ykq.png align="left")

## How ‘title\_\_icontains' works.

I searched for bots ant the queryset returned

![Search results screen when searched bots](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/83kh1brer0msf71rc6zt.png align="left")

But If I search `python bots`

![Search results screen when searched python bots](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mns63lxgtoy1tj6p2gbj.png align="left")

No result since I have no specific term 'python bots' in the title of the post.

Let's correct that with the full text search.

# How to use PostgreSQL in Django?

Full text is only possible in an advance production ready database like PostgreSQL. We can use the PostgreSQL database within our Django app by downloading the database locally or using an online database as a service. For this tutorial we will use a service called [ElephantSQL](https://www.elephantsql.com/), a PostgreSQL as a Service.

Steps we have to follow:

* Create a remote PostgreSQL Database
    
* Connect this Database to Django project.
    
    * Change settings to include postgresql db backend
        
    * Install psycopg2 to talk with PostgreSQL using Python
        

Create a fresh account on this website and then select Tiny Turtle Plan (Free/Slow). Once you create a new instance you will get this page.

![Instance Screen in ElephantSQL](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hqcm0o9ijk7c91jltu08.png align="left")

Copy the URL because that is the only thing you need to connect this database to your local Django Application.

Make the following changes in your settings file core/settings.py

```plaintext
.....

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django.contrib.postgres",                  #THIS APP
    "blog",
]
.....
DATABASES = {
    "default": {
        # My URL
        # postgres://tdjcpial:7UglyA2MM16ksuez2ICeDxfxZjlQnj9X@salt.db.elephantsql.com/tdjcpial
        # Change the below settings according to your Copied URL.
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "tdjcpial",
        "USER": "tdjcpial",
        "PASSWORD": "7UglyA2MM16ksuez2ICeDxfxZjlQnj9X",
        "HOST": "salt.db.elephantsql.com",
    }
}
.....
```

After this change we need to install db adapter psycopg2 using `pip install psycopg2`

Now our database is fully connected, to use this data base from the admin site run `python manage.py migrate` and then `python manage.py createsuperuser`

You can now create new posts in this database to test the full-text functionality.

## To achieve full-text search we need to do the following steps:

* Parsing Query
    
* Parsing Content/Data
    

open blog/views.py and add the following code then we will go through it line by line.

```plaintext
from django.shortcuts import render
from .models import Post
from django.contrib.postgres.search import SearchVector, SearchQuery

# Create your views here.


def home(request):
    qs = Post.objects.all()
    query = request.GET.get("query")
    if query:
        # qs = Post.objects.filter(title__icontains=query)
        qs = Post.objects.annotate(search=SearchVector("title", "content")).filter(search=SearchQuery(query))
    return render(request, "index.html", context={"queryset": qs})
```

In the above code we commented out i\_contains, and added full-text search class provided by the Django PostgreSQL backend.

Let's understand what each class is doing:

* SearchVector(\*fields) : This class processes the data from which the query needs to be searched, this class runs ts\_vector function form the PostgreSQL to break down words into tokens and assigning type to each token.
    
* SearchQuery(query) : This class processes the user provided query to achieve full-text search by breaking query into tokens and removing unnecessary parts from the query like punctuation marks(, ! ?) and words like this(and, or, either).
    

The annotate function creates a new temporary column in a table with values SearchVector(title, content), which means that we want to search both the fields for our query. The filter is then filter out those rows which has the query in their search column.

Let's test what we get after adding this change to our view.

![Search results screen when searched python bots using full text saerch](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hjgvxdgl5ffh62205ksz.png align="left")

VOILA! Remember earlier we were not getting any results on `python bots` but now our app is showing some relevant post to our query.

# There's more to it:

### SearchHeadline() :

This class is used to highlight the results if your query matches with the content, the first parameter determines where we need to highlight in this case, the content, the second parameter takes SearchQuery(query), which we already discussed and the third and fourth shows when the matching tokens are found then how should we enclose them, in this case we used bold, italic and underline tags. Add the following code to your blog/views.py

```plaintext
from django.shortcuts import render
from .models import Post
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchHeadline

## Create your views here.


def home(request):
    qs = Post.objects.all()
    query = request.GET.get("query")
    if query:
        # qs = Post.objects.filter(title__icontains=query)
        # qs = Post.objects.annotate(search=SearchVector("title", "content")).filter(
        #     search=SearchQuery(query)
        # )
        qs = Post.objects.annotate(
            headline=SearchHeadline(
                "content",
                SearchQuery(query),
                start_sel="<b><u><i>",
                stop_sel="</i></u></b>",
            )
        )
    return render(request, "index.html", context={"queryset": qs})
```

For this to work we need to change our template file just a bit.

```plaintext
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Full Text Search</title>
  </head>
  <body>
      <form>
           {% csrf_token %}
          <input type="text" name="query">
          <input type="submit" value="Search">
    <form/>
    {% for object in queryset %}
    <h2>{{object.title}}</h2>
    <p>{{object.headline | safe}}</p>
    {% empty %}
    <h2>No Results</h2>
    {% endfor %}
  </body>
</html>
```

Lets take a look:

![full text search headline example](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fid0g9xrj114pjck6npa.png align="left")

We changed `{{object.content}}` to `{{object.headline}}` and added safe template tag filter so our HTML `<b><u><i>query</i></u></b>` get executed.

See how our query get a bold and itallic.

# Conclusion

For any further information and documentation:

* [Django Full Text](https://docs.djangoproject.com/en/4.0/ref/contrib/postgres/search/)
    
* [PostgreSQL Documentation](https://www.postgresql.org/docs/current/textsearch-controls.html)
    

[GitHub Code Repository](https://github.com/vivekthedev/full-text-search-blog-example), I'm not changing the settings of my database, in case you want to fiddle with the data.

If you liked the article also consider checking me out on [Twitter](https://twitter.com/vivekthedev/) where I post stuff like this in under 280 characters daily.