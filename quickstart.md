# Quickstart

- [A Minimal Application](#a-minimal-application)
- [What to do if the Server does not Start](#what-to-do-if-the-server-does-not-start)
- [Debug Mode](#debug-mode)
- [Routing](#routing)
- [Static Files](#static-files)
- [Rendering Templates](#rendering-templates)
- [Accessing Request Data](#accessing-request-data)
- [Redirects and Errors](#redirect-and-errors)
- [About Responses](#about-responses)
- [Sessions](#sessions)
- [Message Flashing](#message-flashing)
- [Logging](#logging)
- [Hooking in WSGI Middlewares](#hooking-in-wsgi-middlewares)
- [Using Flask Extensions](#using-flask-extensions)
- [Deploying to a Web Server](#deploying-to-a-web-server)

Eager to get started? This page gives a good introduction to Flask. It assumes you
already have Flask installed. If you do not, head over to the [Installation](/docs/{{version}}/installation) section.

<a name="a-minimal-application"></a>
## A Minimal Application

A minimal Flask application looks something like this:

    from flask import Flask
    app = Flask(__name__)

    @app.route('/')
    def hello_world():
        return 'Hello, World!'

So what did that code do?

1. First we imported the **Flask** class. An instance of this class will be our WSGI application.

2. Next we create an instance of this class. The first argument is the name of the application's module or package. If you are using a single module (as in this example), you should use `__name__` because depending on if it's started as application or imported as module the name will be different (`'__main__'` versus the actual import name). This is needed so that Flask knows where to look for templates, static files, and so on. For more information have a look at the **Flask** documentation.

3. We then use the **route()** decorator to tell Flask what URL should trigger our function.

4. The function is given a name which is also used to generate URLs for that particular function, and returns the message we want to display in the user's browser.

Just save it as `hello.py` or something similar. Make sure to not call your application `flask.py` because this would conflict with Flask itself.

To run the application you can either use the **flask** command or python's `-m` switch with Flask. Before you can do that you need to tell your terminal the application to work with by exporting the `FLASK_APP` environment variable:

    $ export FLASK_APP=hello.py
    $ flask run
     * Running on http://127.0.0.1:5000/

If you are on Windows you need to use `set` instead of `export`.

Alternatively you can use `python -m flask`:

    $ export FLASK_APP=hello.py
    $ python -m flask run
     * Running on http://127.0.0.1:5000/

This launches a very simple builtin server, which is good enough for testing but probably not what you want to use in production. For deployment options see [Deployment Options](/docs/{{version}}/deploying).

Now head over to [http://127.0.0.1:5000/](http://127.0.0.1:5000/), and you should see your hello world greeting.

### Externally Visible Server:

If you run the server you will notice that the server is only accessible from your own computer, not from any other in the network. This is the default because in debugging mode a user of the application can execute arbitrary Python code on your computer.

If you have the debugger disabled or trust the users on your network, you can make the server publicly available simply by adding `--host=0.0.0.0` to the command line:

    flask run --host=0.0.0.0

This tells your operating system to listen on all public IPs.

<a name="what-to-do-if-the-server-does-not-start"></a>
## What to do if the Server does not Start

In case the `python -m flask` fails or `flask` does not exist, there are multiple reasons this might be the case. First of all you need to look at the error message.

### Old Version of Flask

Versions of Flask older than 0.11 use to have different ways to start the application. In short, the `flask` command did not exist, and neither did `python -m flask`. In that case you have two options: either upgrade to newer Flask versions or have a look at the [Development Server](/docs/{{version}}/server) docs to see the alternative method for running a server.

### Invalid Import Name

The `FLASK_APP` environment variable is the name of the module to import at `flask run`. In case that module is incorrectly named you will get an import error upon start (or if debug is enabled when you navigate to the application). It will tell you what it tried to import and why it failed.

The most common reason is a typo or because you did not actually create an `app` object.

<a name="debug-mode"></a>
## Debug Mode

(Want to just log errors and stack traces? See [Application Errors](/docs/{{version}}/errors))

The `flask` script is nice to start a local development server, but you would have to restart it manually after each change to your code. That is not very nice and Flask can do better. If you enable debug support the server will reload itself on code changes, and it will also provide you with a helpful debugger if things go wrong.

To enable debug mode you can export the `FLASK_DEBUG` environment variable before running the server:

    $ export FLASK_DEBUG=1
    $ flask run

(On Windows you need to use `set` instead of `export`).

This does the following things:

1. it activates the debugger

2. it activates the automatic reloader

3. it enables the debug mode on the Flask application.

There are more parameters that are explained in the [Development Server](/docs/{{version}}/server) docs.

> {note} Even though the interactive debugger does not work in forking environments (which makes it nearly impossible to use on production servers), it still allows the execution of arbitrary code. This makes it a major security risk and therefore it **must never be used on production machines**.

Have another debugger in mind? See [Working with Debuggers](/docs/{{version}}/errors).

<a name="routing"></a>
## Routing

Modern web applications have beautiful URLs. This helps people remember the URLs, which is especially handy for applications that are used from mobile devices with slower network connections. If the user can directly go to the desired page without having to hit the index page it is more likely they will like the page and come back next time.

As you have seen above, the **route()** decorator is used to bind a function to a URL. Here are some basic examples:

    @app.route('/')
    def index():
        return 'Index Page'

    @app.route('/hello')
    def hello():
        return 'Hello, World'

But there is more to it! You can make certain parts of the URL dynamic and attach multiple rules to a function.

### Variable Rules

To add variable parts to a URL you can mark these special sections as `<variable_name>`. Such a part is then passed as a keyword argument to your function. Optionally a converter can be used by specifying a rule with `<converter:variable_name>`. Here are some nice examples:

    @app.route('/user/<username>')
    def show_user_profile(username):
        # show the user profile for that user
        return 'User %s' % username

    @app.route('/post/<int:post_id>')
    def show_post(post_id):
        # show the post with the given id, the id is an integer
        return 'Post %d' % post_id

The following converters exist:

string  accepts any text without a slash (the default)
int accepts integers
float   like int but for floating point values
path    like the default but also accepts slashes
any matches one of the items provided
uuid    accepts UUID strings

### Unique URLs / Redirection Behavior

Flask's URL rules are based on Werkzeug's routing module. The idea behind that module is to ensure beautiful and unique URLs based on precedents laid down by Apache and earlier HTTP servers.

Take these two rules:

    @app.route('/projects/')
    def projects():
        return 'The project page'

    @app.route('/about')
    def about():
        return 'The about page'

Though they look rather similar, they differ in their use of the trailing slash in the URL definition. In the first case, the canonical URL for the `projects` endpoint has a trailing slash. In that sense, it is similar to a folder on a filesystem. Accessing it without a trailing slash will cause Flask to redirect to the canonical URL with the trailing slash.

In the second case, however, the URL is defined without a trailing slash, rather like the pathname of a file on UNIX-like systems. Accessing the URL with a trailing slash will produce a 404 "Not Found" error.

This behavior allows relative URLs to continue working even if the trailing slash is omitted, consistent with how Apache and other servers work. Also, the URLs will stay unique, which helps search engines avoid indexing the same page twice.

### URL Building

If it can match URLs, can Flask also generate them? Of course it can. To build a URL to a specific function you can use the **url_for()** function. It accepts the name of the function as first argument and a number of keyword arguments, each corresponding to the variable part of the URL rule. Unknown variable parts are appended to the URL as query parameters. Here are some examples:

    >>> from flask import Flask, url_for
    >>> app = Flask(__name__)
    >>> @app.route('/')
    ... def index(): pass
    ...
    >>> @app.route('/login')
    ... def login(): pass
    ...
    >>> @app.route('/user/<username>')
    ... def profile(username): pass
    ...
    >>> with app.test_request_context():
    ...  print url_for('index')
    ...  print url_for('login')
    ...  print url_for('login', next='/')
    ...  print url_for('profile', username='John Doe')
    ...
    /
    /login
    /login?next=/
    /user/John%20Doe

(This also uses the **test_request_context()** method, explained below. It tells Flask to behave as though it is handling a request, even though we are interacting with it through a Python shell. Have a look at the explanation below. [Context Locals](/docs/{{version}}/quickstart#context-locals)).

Why would you want to build URLs using the URL reversing function **url_for()** instead of hard-coding them into your templates? There are three good reasons for this:

1. Reversing is often more descriptive than hard-coding the URLs. More importantly, it allows you to change URLs in one go, without having to remember to change URLs all over the place.

2. URL building will handle escaping of special characters and Unicode data transparently for you, so you don't have to deal with them.

3. If your application is placed outside the URL root - say, in `/myapplication` instead of `/` - **url_for()** will handle that properly for you.

### HTTP Methods

HTTP (the protocol web applications are speaking) knows different methods for accessing URLs. By default, a route only answers to `GET` requests, but that can be changed by providing the `methods` argument to the **route()** decorator. Here are some examples:

    from flask import request

    @app.route('/login', methods=['GET', 'POST'])
    def login():
        if request.method == 'POST':
            do_the_login()
        else:
            show_the_login_form()

If `GET` is present, `HEAD` will be added automatically for you. You don't have to deal with that. It will also make sure that `HEAD` requests are handled as the [HTTP RFC](http://www.ietf.org/rfc/rfc2068.txt) (the document describing the HTTP protocol) demands, so you can completely ignore that part of the HTTP specification. Likewise, as of Flask 0.6, `OPTIONS` is implemented for you automatically as well.

You have no idea what an HTTP method is? Worry not, here is a quick introduction to HTTP methods and why they matter:

The HTTP method (also often called "the verb") tells the server what the client wants to do with the requested page. The following methods are very common:

`GET`

The browser tells the server to just get the information stored on that page and send it. This is probably the most common method.

`HEAD`

The browser tells the server to get the information, but it is only interested in the headers, not the content of the page. An application is supposed to handle that as if a GET request was received but to not deliver the actual content. In Flask you don't have to deal with that at all, the underlying Werkzeug library handles that for you.

`POST`

The browser tells the server that it wants to post some new information to that URL and that the server must ensure the data is stored and only stored once. This is how HTML forms usually transmit data to the server.

`PUT`

Similar to POST but the server might trigger the store procedure multiple times by overwriting the old values more than once. Now you might be asking why this is useful, but there are some good reasons to do it this way. Consider that the connection is lost during transmission: in this situation a system between the browser and the server might receive the request safely a second time without breaking things. With POST that would not be possible because it must only be triggered once.

`DELETE`

Remove the information at the given location.

`OPTIONS`

Provides a quick way for a client to figure out which methods are supported by this URL. Starting with Flask 0.6, this is implemented for you automatically.

Now the interesting part is that in HTML4 and XHTML1, the only methods a form can submit to the server are GET and POST. But with JavaScript and future HTML standards you can use the other methods as well. Furthermore HTTP has become quite popular lately and browsers are no longer the only clients that are using HTTP. For instance, many revision control systems use it.

<a name="static-files"></a>
## Static Files

Dynamic web applications also need static files. That's usually where the CSS and JavaScript files are coming from. Ideally your web server is configured to serve them for you, but during development Flask can do that as well. Just create a folder called `static` in your package or next to your module and it will be available at `/static` on the application.

To generate URLs for static files, use the special `'static'` endpoint name:

    url_for('static', filename='style.css')

The file has to be stored on the filesystem as `static/style.css`.

<a name="rendering-templates"></a>
## Rendering Templates

Generating HTML from within Python is not fun, and actually pretty cumbersome because you have to do the HTML escaping on your own to keep the application secure. Because of that Flask configures the [Jinja2](http://jinja.pocoo.org/) template engine for you automatically.

To render a template you can use the **render_template()** method. All you have to do is provide the name of the template and the variables you want to pass to the template engine as keyword arguments. Here's a simple example of how to render a template:

    from flask import render_template

    @app.route('/hello/')
    @app.route('/hello/<name>')
    def hello(name=None):
        return render_template('hello.html', name=name)

Flask will look for templates in the `templates` folder. So if your application is a module, this folder is next to that module, if it's a package it's actually inside your package:

**Case 1**: a module:

    /application.py
    /templates
        /hello.html

**Case 2**: a package:

    /application
        /__init__.py
        /templates
            /hello.html

For templates you can use the full power of Jinja2 templates. Head over to the official [Jinja2 Template Documentation](http://jinja.pocoo.org/docs/templates) for more information.

Here is an example template:

    <!doctype html>
    <title>Hello from Flask</title>
    {% if name %}
      <h1>Hello {{ name }}!</h1>
    {% else %}
      <h1>Hello, World!</h1>
    {% endif %}

Inside templates you also have access to the **request**, **session** and **g** [1] objects as well as the **get_flashed_messages()** function.

Templates are especially useful if inheritance is used. If you want to know how that works, head over to the [Template Inheritance](/docs/{{version}}/template-inheritance) pattern documentation. Basically template inheritance makes it possible to keep certain elements on each page (like header, navigation and footer).

Automatic escaping is enabled, so if `name` contains HTML it will be escaped automatically. If you can trust a variable and you know that it will be safe HTML (for example because it came from a module that converts wiki markup to HTML) you can mark it as safe by using the **Markup** class or by using the `|safe` filter in the template. Head over to the Jinja 2 documentation for more examples.

Here is a basic introduction to how the **Markup** class works:

    >>> from flask import Markup
    >>> Markup('<strong>Hello %s!</strong>') % '<blink>hacker</blink>'
    Markup(u'<strong>Hello &lt;blink&gt;hacker&lt;/blink&gt;!</strong>')
    >>> Markup.escape('<blink>hacker</blink>')
    Markup(u'&lt;blink&gt;hacker&lt;/blink&gt;')
    >>> Markup('<em>Marked up</em> &raquo; HTML').striptags()
    u'Marked up \xbb HTML'

Changed in version 0.5: Autoescaping is no longer enabled for all templates. The following extensions for templates trigger autoescaping: `.html`, `.htm`, `.xml`, `.xhtml`. Templates loaded from a string will have autoescaping disabled.

> {tip} [1] Unsure what that **g** object is? It's something in which you can store information for your own needs, check the documentation of that object (**g**) and the [Using SQLite 3 with Flask](/docs/{{version}}/sqlite3) for more information.
