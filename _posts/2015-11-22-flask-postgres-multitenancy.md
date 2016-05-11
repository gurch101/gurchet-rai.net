---
layout: post
title: Leveraging Postgresql Schemas for Multitenancy
summary: Segregating web application data by company using Flask and Postgresql
date: 2015-11-22
tags: python flask postgresql 
---

I'm currently working on a Flask web application that'll be used by multiple companies.
Since data across each company needs to be segregated, I used postgresql schemas
to make it all work. 

### What's a Schema?

A postgresql database has one or more schemas which, in turn, contain one or more postgresql objects (tables, procedures, types, etc). Schemas effectively serve
as a namespace for objects in a database. When issuing a query, you can either use `<tablename>` or `<schemaname>.<tablename>`. 

### The Schema Search Path

When using an unqualified name (ie - `<tablename>`), the system looks for
the table on the schema search path and issues queries using the first match.

The current search path can be shown with the following command:
{% highlight sql %}
SHOW search_path;
{% endhighlight %}

By default, this returns:
{% highlight sql %}
  search_path
--------------
 $user,public
{% endhighlight %}

The first member of the default search path is the current user name, the second
member is `public` which is the schema to which tables are added by default.

To update the schema, we can update the schema path with the following command:

{% highlight sql %}
SET search_path TO companyname;
{% endhighlight %}

### Determining the Users Company

Leveraging schemas and the schema search path provides an easy way to segregate
user data by company. All that remains is coming up with a way to determine the
users company on each request. There are several options:

  1. Make the user enter the company name on login.
  2. Store the users and the users company information in the 'public' schema
  3. Use subdomains which contain the company name
    
In the example below, we'll use option 2. Next week, I'll write up a post on how
to use option 3 with Flask. 

### An Example App

{% highlight python %}
from flask import Flask, g, session, request, jsonify, abort
from passlib.hash import pbkdf2_sha256
from psycopg2.pool import ThreadedConnectionPool
from psycopg2.extras import RealDictCursor
from functools import wraps

app = Flask(__name__)
app.secret_key = '\xbc\xd7S\x07\x08\xe9H\x91\xdb\x8c\xdc!\x11\x0f\t\xfe\x9b \xb3so\xd8|]'

pool = ThreadedConnectionPool(1,20,
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
                              
@app.before_request
def start():
    g.db = pool.getconn()
    g.user = session.get('user', None)
    if 'site' in session:        
        with g.db.cursor() as cur:
            cur.execute('SET search_path TO %s', (session['site'],))
            
            
@app.teardown_request
def end(exception):
    db = getattr(g, 'db', None)
    if db is not None:
        pool.putconn(db)
                  
                                   
@app.route('/login', methods=['POST'])
def login():
    username = request.form.get('uname', '')
    password = request.form.get('passwd', '')
    with g.db.cursor() as cur:
        cur.execute('SELECT * from app_user,company \
                      WHERE username=%s \
                        AND company.id = app_user.company_id', (username,))
        user = cur.fetchone()
        if user is not None and pbkdf2_sha256.verify(password, user['password']):
            session['user'] = user['username']
            session['site'] = user['company_name']
            return jsonify(msg='login successful'), 200
        abort(401)


@app.route('/logout', methods=['POST'])    
def logout():
    session.pop('user', None)
    return jsonify(msg='logout successful'), 200
    

@app.route('/data', methods=['GET'])    
@login_required
def get_data():
    with g.db.cursor() as cur:
        cur.execute('SELECT * FROM company_data')           
        return jsonify(data=cur.fetchall()), 200
    
        
if __name__ == "__main__":
    app.run(debug=True)
{% endhighlight %}

#### Schema and Test Data
{% highlight sql %}
CREATE TABLE company (
    id SERIAL PRIMARY KEY,
    company_name TEXT 
);

CREATE TABLE app_user (
    id SERIAL PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL,
    company_id INT REFERENCES company (id)
);

CREATE SCHEMA "company1";
CREATE SCHEMA "company2";

CREATE TABLE company1.company_data (
    id SERIAL PRIMARY KEY,
    description TEXT NOT NULL
); 

CREATE TABLE company2.company_data (
    id SERIAL PRIMARY KEY,
    description TEXT NOT NULL
); 

INSERT INTO company(company_name) VALUES ('company1');
INSERT INTO company(company_name) VALUES ('company2');

# password is 'foo'
INSERT INTO app_user(username, password, company_id) VALUES ('user_1', '$pbkdf2-sha256$29000$5ry31vofg7CWkhJCSClFKA$i01NZ9cAJCAYlXQCY2AXmcmJfe8eD5vZMDOy0h8tH2U', 1);

# password is 'foo'
INSERT INTO app_user(username, password, company_id) VALUES ('user_2', '$pbkdf2-sha256$29000$5ry31vofg7CWkhJCSClFKA$i01NZ9cAJCAYlXQCY2AXmcmJfe8eD5vZMDOy0h8tH2U', 2);

INSERT INTO company1.company_data(description) VALUES ('company 1 data');
INSERT INTO company2.company_data(description) VALUES ('company 2 data');
{% endhighlight %}

### Verifying Behavior with curl
Logging in:
{% highlight bash %}
curl -c - --data "uname=user_1&passwd=foo" http://localhost:5000/login > cookie.txt
{% endhighlight %}
getting data:
{% highlight bash %}
curl -b cookie.txt http://localhost:5000/data
{
  "data": [
    {
      "description": "company 1 data",
      "id": 1
    }
  ]
}
{% endhighlight %}
