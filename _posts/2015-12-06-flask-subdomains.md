---
layout: post
title: Subdomains in Flask
summary: Using subdomains to identify tenants in a multitenant Flask application 
date: 2015-12-06
tags: python flask postgresql
---

In my last post, I wrote about [using postgresql schemas to support multiple tenants from a single database](http://gurchet-rai.net/flask-postgres-multitenancy.html). To identify a tenant, we relied on a tenant identifier to be present in the user record itself. This week, we'll use subdomains to identify the tenant.

### Making Flask Play Nice with Subdomains
At a bare minimum, there are only two things that need to be done in order to make Flask work with subdomains:

1. set the `SERVER_NAME` config value to `<hostname>:<port>`. By default, session cookies will be valid on all subdomains of `SERVER_NAME` 
2. set the `subdomain` parameter on any url rules or blueprints. The parameter can be either static (`subdomain='foo'`) or dynamic (`subdomain='<tenant>'`). 

### Dealing with Static Resources
If you're using Flask to serve static resources rather than a web server, you'll need to manually register the static folder url rule so that you can configure it to support subdomains. Here's how you do that:

{% highlight python %}
app = Flask(__name__, static_folder=None)
app.static_folder='static'
app.add_url_rule('/static/<path:filename>',
                 endpoint='static',
                 subdomain='<tenant>',
                 view_func=app.send_static_file)                 

# optional. If not set, the above view_func will be passed <tenant> as a parameter.
@app.url_value_preprocessor
def before_route(endpoint, values):
    if values is not None:
        values.pop('tenant', None)

{% endhighlight %}

With the above, static resources will be accessible from one central location, regardless of subdomain.

### Testing in a Development Environment
Flask doesn't support subdomains on `localhost` or on host names without a tld identifier. For the example app below, I added the following entry to `/etc/hosts`:
    
{% highlight bash %}
127.0.0.1 local.com
127.0.0.1 company1.local.com
127.0.0.1 company2.local.com
{% endhighlight %}

### An Example App
{% highlight python %}
from functools import wraps
from urlparse import urlparse
from flask import Flask, g, session, request, abort, jsonify
from psycopg2.pool import ThreadedConnectionPool
from psycopg2.extras import RealDictCursor
from passlib.hash import pbkdf2_sha256


app = Flask(__name__, static_folder=None)
app.static_folder = 'static'
app.add_url_rule('/static/<path:filename>',
                 endpoint='static',
                 subdomain='<tenant>',
                 view_func=app.send_static_file)
app.secret_key = ('\xbc\xd7S\x07\x08\xe9H\x91\xdb\x8c'
                  '\xdc!\x11\x0f\t\xfe\x9b \xb3so\xd8|]')
# IMPORTANT! subdomains will not work without the SERVER_NAME config
app.config['SERVER_NAME'] = 'local.com:5000'

pool = ThreadedConnectionPool(1, 20,
                              host='127.0.0.1',
                              database='test',
                              user='test',
                              password='test',
                              cursor_factory=RealDictCursor)


def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if g.user is None:
            abort(401)
        return f(*args, **kwargs)
    return decorated_function


@app.url_value_preprocessor
def before_route(endpoint, values):
    # most of our endpoints don't care about the subdomain
    # so remove it from the set of parameters passed
    # to the route
    if (endpoint is not 'login' and
            values is not None):
        values.pop('tenant', None)


def schema_exists(schema_name):
    with g.db.cursor() as cur:
        cur.execute(('select nspname '
                     'from pg_catalog.pg_namespace '
                     'where nspname=%s'), (schema_name,))
        return cur.fetchone() is not None


@app.before_request
def start():
    """init globals and set the schema search path for the current request. """
    g.db = pool.getconn()
    g.user = session.get('user', None)
    site = session.get('site', None)
    subdomain = urlparse(request.url).hostname.split('.')[0]

    if request.endpoint == 'login':
        site = subdomain
        if not schema_exists(site):
            abort(400)

    if site != subdomain:
        abort(400)

    with g.db.cursor() as cur:
        cur.execute('SET search_path TO %s', (site,))


@app.teardown_request
def end(exception):
    db = getattr(g, 'db', None)
    if db is not None:
        pool.putconn(db)


@app.route('/login', methods=['POST'], subdomain='<tenant>')
def login(tenant):
    username = request.form.get('uname', '')
    password = request.form.get('passwd', '')
    with g.db.cursor() as cur:
        cur.execute('SELECT * from app_user \
                     WHERE username=%s', (username,))
        user = cur.fetchone()
        if (user is not None and
                pbkdf2_sha256.verify(password, user['password'])):
            session['user'] = user['username']
            session['site'] = tenant
            return jsonify(msg='login successful'), 200
        abort(401)


@app.route('/logout', methods=['POST'], subdomain='<tenant>')
def logout():
    session.pop('user', None)
    return jsonify(msg='logout successful'), 200


@app.route('/data', methods=['GET'], subdomain='<tenant>')
@login_required
def get_data():
    with g.db.cursor() as cur:
        cur.execute('SELECT * FROM company_data')
        return jsonify(data=cur.fetchall()), 200


if __name__ == '__main__':
    app.run(debug=True)
{% endhighlight %}

#### Schema and Test Data
{% highlight sql %}
CREATE SCHEMA "company1";
CREATE SCHEMA "company2";

CREATE TABLE company1.app_user (
    id SERIAL PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
);

CREATE TABLE company2.app_user (
    id SERIAL PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
);

CREATE TABLE company1.company_data (
    id SERIAL PRIMARY KEY,
    description TEXT NOT NULL
); 

CREATE TABLE company2.company_data (
    id SERIAL PRIMARY KEY,
    description TEXT NOT NULL
); 

INSERT INTO company1.app_user(username, password) VALUES ('user_1', '$pbkdf2-sha256$29000$5ry31vofg7CWkhJCSClFKA$i01NZ9cAJCAYlXQCY2AXmcmJfe8eD5vZMDOy0h8tH2U');

INSERT INTO company2.app_user(username, password) VALUES ('user_2', '$pbkdf2-sha256$29000$5ry31vofg7CWkhJCSClFKA$i01NZ9cAJCAYlXQCY2AXmcmJfe8eD5vZMDOy0h8tH2U');

INSERT INTO company1.company_data(description) VALUES ('company 1 data');
INSERT INTO company2.company_data(description) VALUES ('company 2 data');
{% endhighlight %}

### Verifying Behaviour with curl
Logging in:
{% highlight bash %}
curl -c - --data "uname=user_1&passwd=foo" http://company1.local.com:5000/login > cookie.txt
{% endhighlight %}
getting data:
{% highlight bash %}
curl -b cookie.txt http://company1.local.com:5000/data
{
  "data": [
    {
      "description": "company 1 data",
      "id": 1
    }
  ]
}
{% endhighlight %}

