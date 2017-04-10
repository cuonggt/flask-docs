# Tutorial

- [Introducing Flaskr](#introducing-flask)
- [Step 0: Creating The Folders](#system-wide-installation)
- [Step 1: Database Schema](#database-schema)
- [Step 2: Application Setup Code](#application-setup-code)
- [Step 3: Installing flaskr as a Package](#installing-flaskr-as-a-package)
- [Step 4: Database Connections](#database-connections)
- [Step 5: Creating The Database](#creating-the-database)
- [Step 6: The View Functions](#the-view-functions)
    - [Show Entries](#show-entries)
    - [Add New Entry](#add-new-entry)
    - [Login and Logout](#login-and-logout)
- [Step 7: The Templates](#the-templates)
    - [layout.html](#layout-html)
    - [show_entries.html](#show-entries-html)
    - [login.html](#login-html)
- [Step 8: Adding Style](#adding-style)
- [Bonus: Testing the Application](#testing-the-application)
    - [Adding tests to flaskr](#adding-tests-to-flaskr)
    - [Running the tests](#running-the-tests)
    - [Testing + setuptools](#testing-setuptools)

<a name="introducing-flask"></a>
## Introducing Flask

This tutorial will demonstrate a blogging application named Flaskr, but feel free to choose your own less Web-2.0-ish name ;) Essentially, it will do the following things:

Let the user sign in and out with credentials specified in the configuration. Only one user is supported.
When the user is logged in, they can add new entries to the page consisting of a text-only title and some HTML for the text. This HTML is not sanitized because we trust the user here.
The index page shows all entries so far in reverse chronological order (newest on top) and the user can add new ones from there if logged in.
SQLite3 will be used directly for this application because it's good enough for an application of this size. For larger applications, however, it makes a lot of sense to use [SQLAlchemy](http://www.sqlalchemy.org/), as it handles database connections in a more intelligent way, allowing you to target different relational databases at once and more. You might also want to consider one of the popular NoSQL databases if your data is more suited for those.

Here a screenshot of the final application:

![screenshot of the final application](http://flask.pocoo.org/docs/0.12/_images/flaskr.png)

Continue with [Step 0: Creating The Folders](#creating-the-folders).

<a name="creating-the-folders"></a>
## Step 0: Creating The Folders

Before getting started, you will need to create the folders needed for this application:

    /flaskr
        /flaskr
            /static
            /templates
The application will be installed and run as Python package. This is the recommended way to install and run Flask applications. You will see exactly how to run `flaskr` later on in this tutorial. For now go ahead and create the applications directory structure. In the next few steps you will be creating the database schema as well as the main module.

As a quick side note, the files inside of the `static` folder are available to users of the application via HTTP. This is the place where CSS and JavaScript files go. Inside the `templates` folder, Flask will look for [Jinja2](http://jinja.pocoo.org/) templates. You will see examples of this later on.

For now you should continue with [Step 1: Database Schema](#database-schema).

<a name="database-schema"></a>
## Step 1: Database Schema

In this step, you will create the database schema. Only a single table is needed for this application and it will only support SQLite. All you need to do is put the following contents into a file named `schema.sql` in the `flaskr/flaskr` folder:

    drop table if exists entries;
    create table entries (
      id integer primary key autoincrement,
      title text not null,
      'text' text not null
    );

This schema consists of a single table called `entries`. Each row in this table has an `id`, a `title`, and a `text`. The `id` is an automatically incrementing integer and a primary key, the other two are strings that must not be null.

Continue with [Step 2: Application Setup Code](#application-setup-code).

<a name="application-setup-code"></a>
## Application Setup Code

Now that the schema is in place, you can create the application module, `flaskr.py`. This file should be placed inside of the `flaskr/flaskr` folder. The first several lines of code in the application module are the needed import statements. After that there will be a few lines of configuration code. For small applications like `flaskr`, it is possible to drop the configuration directly into the module. However, a cleaner solution is to create a separate `.ini` or `.py` file, load that, and import the values from there.

Here are the import statements (in `flaskr.py`):

    # all the imports
    import os
    import sqlite3
    from flask import Flask, request, session, g, redirect, url_for, abort, \
         render_template, flash

The next couple lines will create the actual application instance and initialize it with the config from the same file in `flaskr.py`:

    app = Flask(__name__) # create the application instance :)
    app.config.from_object(__name__) # load config from this file , flaskr.py

    # Load default config and override config from an environment variable
    app.config.update(dict(
        DATABASE=os.path.join(app.root_path, 'flaskr.db'),
        SECRET_KEY='development key',
        USERNAME='admin',
        PASSWORD='default'
    ))
    app.config.from_envvar('FLASKR_SETTINGS', silent=True)

The **Config** object works similarly to a dictionary, so it can be updated with new values.

> **Database Path**

> Operating systems know the concept of a current working directory for each process. Unfortunately, you cannot depend on this in web applications because you might have more than one application in the same process.

> For this reason the `app.root_path` attribute can be used to get the path to the application. Together with the `os.path` module, files can then easily be found. In this example, we place the database right next to it.

> For a real-world application, it's recommended to use [Instance Folders](/docs/{{version}}/configuration#instance-folders) instead.

Usually, it is a good idea to load a separate, environment-specific configuration file. Flask allows you to import multiple configurations and it will use the setting defined in the last import. This enables robust configuration setups. **from_envvar()** can help achieve this.

    app.config.from_envvar('FLASKR_SETTINGS', silent=True)

Simply define the environment variable **FLASKR_SETTINGS** that points to a config file to be loaded. The silent switch just tells Flask to not complain if no such environment key is set.

In addition to that, you can use the **from_object()** method on the config object and provide it with an import name of a module. Flask will then initialize the variable from that module. Note that in all cases, only variable names that are uppercase are considered.

The `SECRET_KEY` is needed to keep the client-side sessions secure. Choose that key wisely and as hard to guess and complex as possible.

Lastly, you will add a method that allows for easy connections to the specified database. This can be used to open a connection on request and also from the interactive Python shell or a script. This will come in handy later. You can create a simple database connection through SQLite and then tell it to use the **sqlite3**.Row object to represent rows. This allows the rows to be treated as if they were dictionaries instead of tuples.

    def connect_db():
        """Connects to the specific database."""
        rv = sqlite3.connect(app.config['DATABASE'])
        rv.row_factory = sqlite3.Row
        return rv

In the next section you will see how to run the application.

Continue with [Step 3: Installing flaskr as a Package](#installing-flaskr-as-a-package).

<a name="installing-flaskr-as-a-package"></a>
## Installing flaskr as a Package

Flask is now shipped with built-in support for [Click](http://click.pocoo.org/). Click provides Flask with enhanced and extensible command line utilities. Later in this tutorial you will see exactly how to extend the `flask` command line interface (CLI).

A useful pattern to manage a Flask application is to install your app following the [Python Packaging Guide](https://packaging.python.org/). Presently this involves creating two new files; `setup.py` and `MANIFEST.in` in the projects root directory. You also need to add an `__init__.py` file to make the `flaskr/flaskr` directory a package. After these changes, your code structure should be:

    /flaskr
        /flaskr
            __init__.py
            /static
            /templates
            flaskr.py
            schema.sql
        setup.py
        MANIFEST.in

The content of the `setup.py` file for `flaskr` is:

    from setuptools import setup

    setup(
        name='flaskr',
        packages=['flaskr'],
        include_package_data=True,
        install_requires=[
            'flask',
        ],
    )

When using setuptools, it is also necessary to specify any special files that should be included in your package (in the `MANIFEST.in`). In this case, the static and templates directories need to be included, as well as the schema. Create the `MANIFEST.in` and add the following lines:

    graft flaskr/templates
    graft flaskr/static
    include flaskr/schema.sql

To simplify locating the application, add the following import statement into this file, `flaskr/__init__.py`:

    from .flaskr import app

This import statement brings the application instance into the top-level of the application package. When it is time to run the application, the Flask development server needs the location of the app instance. This import statement simplifies the location process. Without it the export statement a few steps below would need to be `export FLASK_APP=flaskr.flaskr`.

At this point you should be able to install the application. As usual, it is recommended to install your Flask application within a [virtualenv](https://virtualenv.pypa.io/). With that said, go ahead and install the application with:

    pip install --editable .

The above installation command assumes that it is run within the projects root directory, *flaskr/*. The *editable* flag allows editing source code without having to reinstall the Flask app each time you make changes. The flaskr app is now installed in your virtualenv (see output of `pip freeze`).

With that out of the way, you should be able to start up the application. Do this with the following commands:

    export FLASK_APP=flaskr
    export FLASK_DEBUG=true
    flask run

(In case you are on Windows you need to use *set* instead of *export*). The **FLASK_DEBUG** flag enables or disables the interactive debugger. *Never leave debug mode activated in a production system*, because it will allow users to execute code on the server!

You will see a message telling you that server has started along with the address at which you can access it.

When you head over to the server in your browser, you will get a 404 error because we don't have any views yet. That will be addressed a little later, but first, you should get the database working.

<a name="#public-server"></a>
> **Externally Visible Server**

> Want your server to be publicly available? Check out the [externally visible server](/docs/{{version}}/quickstart#public-server) section for more information.

Continue with [Step 4: Database Connections](#database-connections).

<a name="database-connections"></a>
## Step 4: Database Connections

You currently have a function for establishing a database connection with *connect_db*, but by itself, it is not particularly useful. Creating and closing database connections all the time is very inefficient, so you will need to keep it around for longer. Because database connections encapsulate a transaction, you will need to make sure that only one request at a time uses the connection. An elegant way to do this is by utilizing the *application context*.

Flask provides two contexts: the *application context* and the *request context*. For the time being, all you have to know is that there are special variables that use these. For instance, the **request** variable is the request object associated with the current request, whereas **g** is a general purpose variable associated with the current application context. The tutorial will cover some more details of this later on.

For the time being, all you have to know is that you can store information safely on the **g** object.

So when do you put it on there? To do that you can make a helper function. The first time the function is called, it will create a database connection for the current context, and successive calls will return the already established connection:

    def get_db():
        """Opens a new database connection if there is none yet for the
        current application context.
        """
        if not hasattr(g, 'sqlite_db'):
            g.sqlite_db = connect_db()
        return g.sqlite_db

Now you know how to connect, but how can you properly disconnect? For that, Flask provides us with the **teardown_appcontext()** decorator. It's executed every time the application context tears down:

    @app.teardown_appcontext
    def close_db(error):
        """Closes the database again at the end of the request."""
        if hasattr(g, 'sqlite_db'):
            g.sqlite_db.close()

Functions marked with **teardown_appcontext()** are called every time the app context tears down. What does this mean? Essentially, the app context is created before the request comes in and is destroyed (torn down) whenever the request finishes. A teardown can happen because of two reasons: either everything went well (the error parameter will be `None`) or an exception happened, in which case the error is passed to the teardown function.

Curious about what these contexts mean? Have a look at the [The Application Context](/docs/{{version}}/app-context) documentation to learn more.

Continue to [Step 5: Creating The Database](#creating-the-database).

> **Where do I put this code?**

> If you've been following along in this tutorial, you might be wondering where to put the code from this step and the next. A logical place is to group these module-level functions together, and put your new `get_db` and `close_db` functions below your existing `connect_db` function (following the tutorial line-by-line).

> If you need a moment to find your bearings, take a look at how the [example source](https://github.com/pallets/flask/tree/master/examples/flaskr/) is organized. In Flask, you can put all of your application code into a single Python module. You don't have to, and if your app [grows larger](/docs/{{version}}/patterns#larger-applications), it's a good idea not to.

<a name="creating-the-database"></a>
## Step 5: Creating The Database

As outlined earlier, Flaskr is a database powered application, and more precisely, it is an application powered by a relational database system. Such systems need a schema that tells them how to store that information. Before starting the server for the first time, it’s important to create that schema.

Such a schema can be created by piping the `schema.sql` file into the *sqlite3* command as follows:

    sqlite3 /tmp/flaskr.db < schema.sql

The downside of this is that it requires the `sqlite3` command to be installed, which is not necessarily the case on every system. This also requires that you provide the path to the database, which can introduce errors. It’s a good idea to add a function that initializes the database for you, to the application.

To do this, you can create a function and hook it into a **flask** command that initializes the database. For now just take a look at the code segment below. A good place to add this function, and command, is just below the *connect_db* function in `flaskr.py`:

    def init_db():
        db = get_db()
        with app.open_resource('schema.sql', mode='r') as f:
            db.cursor().executescript(f.read())
        db.commit()

    @app.cli.command('initdb')
    def initdb_command():
        """Initializes the database."""
        init_db()
        print('Initialized the database.')

The `app.cli.command()` decorator registers a new command with the **flask** script. When the command executes, Flask will automatically create an application context which is bound to the right application. Within the function, you can then access **flask.g** and other things as you might expect. When the script ends, the application context tears down and the database connection is released.

You will want to keep an actual function around that initializes the database, though, so that we can easily create databases in unit tests later on. (For more information see [Testing Flask Applications](/docs/{{version}}/testing).)

The **open_resource()** method of the application object is a convenient helper function that will open a resource that the application provides. This function opens a file from the resource location (the `flaskr/flaskr` folder) and allows you to read from it. It is used in this example to execute a script on the database connection.

The connection object provided by SQLite can give you a cursor object. On that cursor, there is a method to execute a complete script. Finally, you only have to commit the changes. SQLite3 and other transactional databases will not commit unless you explicitly tell it to.

Now, it is possible to create a database with the **flask** script:

    flask initdb
    Initialized the database.

> **Troubleshooting**

> If you get an exception later on stating that a table cannot be found, check that you did execute the `initdb` command and that your table names are correct (singular vs. plural, for example).

Continue with [Step 6: The View Functions](#the-view-functions).

<a name="the-view-functions"></a>
## The View Functions

Now that the database connections are working, you can start writing the view functions. You will need four of them:

<a name="show-entries"></a>
### Show Entries

This view shows all the entries stored in the database. It listens on the root of the application and will select title and text from the database. The one with the highest id (the newest entry) will be on top. The rows returned from the cursor look a bit like dictionaries because we are using the **sqlite3.Row** row factory.

The view function will pass the entries to the `show_entries.html` template and return the rendered one:

    @app.route('/')
    def show_entries():
        db = get_db()
        cur = db.execute('select title, text from entries order by id desc')
        entries = cur.fetchall()
        return render_template('show_entries.html', entries=entries)

### Add New Entry

This view lets the user add new entries if they are logged in. This only responds to `POST` requests; the actual form is shown on the *show_entries* page. If everything worked out well, it will **flash()** an information message to the next request and redirect back to the *show_entries* page:

    @app.route('/add', methods=['POST'])
    def add_entry():
        if not session.get('logged_in'):
            abort(401)
        db = get_db()
        db.execute('insert into entries (title, text) values (?, ?)',
                     [request.form['title'], request.form['text']])
        db.commit()
        flash('New entry was successfully posted')
        return redirect(url_for('show_entries'))

Note that this view checks that the user is logged in (that is, if the *logged_in* key is present in the session and `True`).

> **Security Note**

> Be sure to use question marks when building SQL statements, as done in the example above. Otherwise, your app will be vulnerable to SQL injection when you use string formatting to build SQL statements. See [Using SQLite 3 with Flask](/docs/{{version}}/patterns#sqlite3) for more.

### Login and Logout

These functions are used to sign the user in and out. Login checks the username and password against the ones from the configuration and sets the *logged_in* key for the session. If the user logged in successfully, that key is set to `True`, and the user is redirected back to the *show_entries* page. In addition, a message is flashed that informs the user that he or she was logged in successfully. If an error occurred, the template is notified about that, and the user is asked again:

    @app.route('/login', methods=['GET', 'POST'])
    def login():
        error = None
        if request.method == 'POST':
            if request.form['username'] != app.config['USERNAME']:
                error = 'Invalid username'
            elif request.form['password'] != app.config['PASSWORD']:
                error = 'Invalid password'
            else:
                session['logged_in'] = True
                flash('You were logged in')
                return redirect(url_for('show_entries'))
        return render_template('login.html', error=error)

The *logout* function, on the other hand, removes that key from the session again. There is a neat trick here: if you use the **pop()** method of the dict and pass a second parameter to it (the default), the method will delete the key from the dictionary if present or do nothing when that key is not in there. This is helpful because now it is not necessary to check if the user was logged in.

    @app.route('/logout')
    def logout():
        session.pop('logged_in', None)
        flash('You were logged out')
        return redirect(url_for('show_entries'))

> **Security Note**

> Passwords should never be stored in plain text in a production system. This tutorial uses plain text passwords for simplicity. If you plan to release a project based off this tutorial out into the world, passwords should be both [hashed and salted](https://blog.codinghorror.com/youre-probably-storing-passwords-incorrectly/) before being stored in a database or file.

> Fortunately, there are Flask extensions for the purpose of hashing passwords and verifying passwords against hashes, so adding this functionality is fairly straight forward. There are also many general python libraries that can be used for hashing.

> You can find a list of recommended Flask extensions [here](http://flask.pocoo.org/extensions/)

Continue with [Step 7: The Templates](#the-templates).

<a name="the-templates"></a>
## Step 7: The Templates

Now it is time to start working on the templates. As you may have noticed, if you make requests with the app running, you will get an exception that Flask cannot find the templates. The templates are using [Jinja2](/docs/{{version}}/jinja) syntax and have autoescaping enabled by default. This means that unless you mark a value in the code with **Markup** or with the `|safe` filter in the template, Jinja2 will ensure that special characters such as `<` or `>` are escaped with their XML equivalents.

We are also using template inheritance which makes it possible to reuse the layout of the website in all pages.

Put the following templates into the `templates` folder:

<a name="layout-html"></a>
### layout.html

This template contains the HTML skeleton, the header and a link to log in (or log out if the user was already logged in). It also displays the flashed messages if there are any. The `{% block body %}` block can be replaced by a block of the same name (`body`) in a child template.

The **session** dict is available in the template as well and you can use that to check if the user is logged in or not. Note that in Jinja you can access missing attributes and items of objects / dicts which makes the following code work, even if there is no `'logged_in'` key in the session:

    <!doctype html>
    <title>Flaskr</title>
    <link rel=stylesheet type=text/css href="{{ url_for('static', filename='style.css') }}">
    <div class=page>
      <h1>Flaskr</h1>
      <div class=metanav>
      {% if not session.logged_in %}
        <a href="{{ url_for('login') }}">log in</a>
      {% else %}
        <a href="{{ url_for('logout') }}">log out</a>
      {% endif %}
      </div>
      {% for message in get_flashed_messages() %}
        <div class=flash>{{ message }}</div>
      {% endfor %}
      {% block body %}{% endblock %}
    </div>

<a name="show-entries-html"></a>
### show_entries.html

This template extends the `layout.html` template from above to display the messages. Note that the `for` loop iterates over the messages we passed in with the **render_template()** function. Notice that the form is configured to to submit to the `add_entry` view function and use `POST` as HTTP method:

    {% extends "layout.html" %}
    {% block body %}
      {% if session.logged_in %}
        <form action="{{ url_for('add_entry') }}" method=post class=add-entry>
          <dl>
            <dt>Title:
            <dd><input type=text size=30 name=title>
            <dt>Text:
            <dd><textarea name=text rows=5 cols=40></textarea>
            <dd><input type=submit value=Share>
          </dl>
        </form>
      {% endif %}
      <ul class=entries>
      {% for entry in entries %}
        <li><h2>{{ entry.title }}</h2>{{ entry.text|safe }}
      {% else %}
        <li><em>Unbelievable.  No entries here so far</em>
      {% endfor %}
      </ul>
    {% endblock %}

<a name="login-html"></a>
### login.html

This is the login template, which basically just displays a form to allow the user to login:

    {% extends "layout.html" %}
    {% block body %}
      <h2>Login</h2>
      {% if error %}<p class=error><strong>Error:</strong> {{ error }}{% endif %}
      <form action="{{ url_for('login') }}" method=post>
        <dl>
          <dt>Username:
          <dd><input type=text name=username>
          <dt>Password:
          <dd><input type=password name=password>
          <dd><input type=submit value=Login>
        </dl>
      </form>
    {% endblock %}

Continue with [Step 8: Adding Style](#adding-style).

<a name="adding-style"></a>
## Step 8: Adding Style

Now that everything else works, it’s time to add some style to the application. Just create a stylesheet called `style.css` in the `static` folder:

    body            { font-family: sans-serif; background: #eee; }
    a, h1, h2       { color: #377ba8; }
    h1, h2          { font-family: 'Georgia', serif; margin: 0; }
    h1              { border-bottom: 2px solid #eee; }
    h2              { font-size: 1.2em; }

    .page           { margin: 2em auto; width: 35em; border: 5px solid #ccc;
                      padding: 0.8em; background: white; }
    .entries        { list-style: none; margin: 0; padding: 0; }
    .entries li     { margin: 0.8em 1.2em; }
    .entries li h2  { margin-left: -1em; }
    .add-entry      { font-size: 0.9em; border-bottom: 1px solid #ccc; }
    .add-entry dl   { font-weight: bold; }
    .metanav        { text-align: right; font-size: 0.8em; padding: 0.3em;
                      margin-bottom: 1em; background: #fafafa; }
    .flash          { background: #cee5F5; padding: 0.5em;
                      border: 1px solid #aacbe2; }
    .error          { background: #f0d6d6; padding: 0.5em; }

Continue with [Bonus: Testing the Application](#testing-the-application).

<a name="testing-the-application"></a>
## Bonus: Testing the Application

Now that you have finished the application and everything works as expected, it’s probably not a bad idea to add automated tests to simplify modifications in the future. The application above is used as a basic example of how to perform unit testing in the [Testing Flask Applications](/docs/{{version}}/testing) section of the documentation. Go there to see how easy it is to test Flask applications.

<a name="adding-tests-to-flaskr"></a>
### Adding tests to flaskr
Assuming you have seen the [Testing Flask Applications](/docs/{{version}}/testing) section and have either written your own tests for `flaskr` or have followed along with the examples provided, you might be wondering about ways to organize the project.

One possible and recommended project structure is:

    flaskr/
        flaskr/
            __init__.py
            static/
            templates/
        tests/
            test_flaskr.py
        setup.py
        MANIFEST.in

For now go ahead a create the `tests/` directory as well as the `test_flaskr.py` file.

<a name="running-the-tests"></a>
### Running the tests

At this point you can run the tests. Here `pytest` will be used.

> {note} Make sure that pytest is installed in the same virtualenv as flaskr. Otherwise pytest test will not be able to import the required components to test the application:

>       pip install -e .
>       pip install pytest

Run and watch the tests pass, within the top-level `flaskr/` directory as:

    py.test

<a name="testing-setuptools"></a>
### Testing + setuptools

One way to handle testing is to integrate it with `setuptools`. Here that requires adding a couple of lines to the `setup.py` file and creating a new file `setup.cfg`. One benefit of running the tests this way is that you do not have to install `pytest`. Go ahead and update the `setup.py` file to contain:

    from setuptools import setup

    setup(
        name='flaskr',
        packages=['flaskr'],
        include_package_data=True,
        install_requires=[
            'flask',
        ],
        setup_requires=[
            'pytest-runner',
        ],
        tests_require=[
            'pytest',
        ],
    )

Now create `setup.cfg` in the project root (alongside `setup.py`):

    [aliases]
    test=pytest

Now you can run:

    python setup.py test

This calls on the alias created in `setup.cfg` which in turn runs `pytest` via `pytest-runner`, as the `setup.py` script has been called. (Recall the *setup_requires* argument in `setup.py`) Following the standard rules of test-discovery your tests will be found, run, and hopefully pass.

This is one possible way to run and manage testing. Here `pytest` is used, but there are other options such as `nose`. Integrating testing with `setuptools` is convenient because it is not necessary to actually download `pytest` or any other testing framework one might use.
