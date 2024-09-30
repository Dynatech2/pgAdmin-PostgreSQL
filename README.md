# pgAdmin-PostgreSQL
## How To Configure Nginx as a Reverse Proxy on Ubuntu 22.04

### Introduction
A reverse proxy is the recommended method to expose an application server to the internet. Whether you are running a Node.js application in production or a minimal built-in web server with Flask, these application servers will often bind to localhost with a TCP port. This means by default, your application will only be accessible locally on the machine it resides on. While you can specify a different bind point to force access through the internet, these application servers are designed to be served from behind a reverse proxy in production environments. This provides security benefits in isolating the application server from direct internet access, the ability to centralize firewall protection, and a minimized attack plane for common threats such as denial of service attacks.

From a client’s perspective, interacting with a reverse proxy is no different from interacting with the application server directly. It is functionally the same, and the client cannot tell the difference. A client requests a resource and then receives it, without any extra configuration required by the client.
This tutorial will demonstrate how to set up a reverse proxy using Nginx, a popular web server and reverse proxy solution. You will install Nginx, configure it as a reverse proxy using the proxy_pass directive, and forward the appropriate headers from your client’s request. If you don’t have an application server on hand to test, you will optionally set up a test application with the WSGI server Gunicorn.
### Prerequisites
To complete this tutorial, you will need:
An Ubuntu 22.04 server, set up according to our initial server setup guide for Ubuntu 22.04,
The address of the application server you want to proxy, this will be referred to as app_server_address throughout the tutorial. This can be an IP address with TCP port (such as the Gunicorn default of http://127.0.0.1:8000), or a unix domain socket (such as http://unix:/tmp/pgadmin4.sock for pgAdmin). If you do not have an application server set up to test with, you will be guided through setting up a Gunicorn application which will bind to http://127.0.0.1:8000.
A domain name pointed at your server’s public IP. This will be configured with Nginx to proxy your application server.
#### Step 1 — Installing Nginx
Nginx is available for installation with apt through the default repositories. Update your repository index, then install Nginx:
```
sudo apt update
sudo apt install nginx
```
Press **Y** to confirm the installation. If you are asked to restart services, press **ENTER** to accept the defaults.
You need to allow access to Nginx through your firewall. Having set up your server according to the initial server prerequisites, add the following rule with ufw:
```
sudo ufw allow 'Nginx HTTP'
```
Now you can verify that Nginx is running:
```
systemctl status nginx
```
<pre>
Output
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-08-29 06:52:46 UTC; 39min ago
       Docs: man:nginx(8)
   Main PID: 9919 (nginx)
      Tasks: 2 (limit: 2327)
     Memory: 2.9M
        CPU: 50ms
     CGroup: /system.slice/nginx.service
             ├─9919 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             └─9920 "nginx: worker process
</pre>
Next you will add a custom server block with your domain and app server proxy.
#### Step 2 — Configuring your Server Block
1. It is recommended practice to create a custom configuration file for your new server block additions, instead of editing the default configuration directly. Create and open a new Nginx configuration file using nano or your preferred text editor:
```
sudo nano /etc/nginx/sites-available/your_domain
```
2. Insert the following into your new file, making sure to replace your_domain and app_server_address. If you do not have an application server to test with, default to using http://127.0.0.1:8000 for the optional Gunicorn server setup in Step 3:
```
/etc/nginx/sites-available/your_domain
server {
    listen 80;
    listen [::]:80;

    server_name your_domain www.your_domain;
        
    location / {
        proxy_pass app_server_address;
        include proxy_params;
    }
}
```
3. Save and exit, with nano you can do this by hitting CTRL+O then CTRL+X.

This configuration file begins with a standard Nginx setup, where Nginx will listen on port 80 and respond to requests made to your_domain and www.your_domain. Reverse proxy functionality is enabled through Nginx’s proxy_pass directive. With this configuration, navigating to your_domain in your local web browser will be the same as opening app_server_address on your remote machine. While this tutorial will only proxy a single application server, Nginx is capable of serving as a proxy for multiple servers at once. By adding more location blocks as needed, a single server name can combine multiple application servers through proxy into one cohesive web application.

All HTTP requests come with headers, which contain information about the client who sent the request. This includes details like IP address, cache preferences, cookie tracking, authorization status, and more. Nginx provides some recommended header forwarding settings you have included as proxy_params, and the details can be found in /etc/nginx/proxy_params:
```
/etc/nginx/proxy_params
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```
With reverse proxies, your goal is to pass on relevant information about the client, and sometimes information about your reverse proxy server itself. There are use cases where a proxied server would want to know which reverse proxy server handled the request, but generally the important information is from the original client’s request. In order to pass on these headers and make information available in locations where it is expected, Nginx uses the proxy_set_header directive.

By default, when Nginx acts as a reverse proxy it alters two headers, strips out all empty headers, then passes on the request. The two altered headers are the Host and Connection header. There are many HTTP headers available, and you can check this detailed list of HTTP headers for more information on each of their purposes, though the relevant ones for reverse proxies will be covered here later.
Here are the headers forwarded by proxy_params and the variables it stores the data in:

- Host: This header contains the original host requested by the client, which is the website domain and port. Nginx keeps this in the $http_host variable.
- X-Forwarded-For: This header contains the IP address of the client who sent the original request. It can also contain a list of IP addresses, with the original client IP coming first, then a list of all the IP addresses of the reverse proxy servers that passed the request through. Nginx keeps this in the $proxy_add_x_forwarded_for variable.
- X-Real-IP: This header always contains a single IP address that belongs to the remote client. This is in contrast to the similar X-Forwarded-For which can contain a list of addresses. If X-Forwarded-For is not present, it will be the same as X-Real-IP.
- X-Forwarded-Proto: This header contains the protocol used by the original client to connect, whether it be HTTP or HTTPS. Nginx keeps this in the $scheme variable.
Next, enable this configuration file by creating a link from it to the sites-enabled directory that Nginx reads at startup:
```
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```
You can now test your configuration file for syntax errors:
```
sudo nginx -t
```
With no problems reported, restart Nginx to apply your changes:
```
sudo systemctl restart nginx
```
Nginx is now configured as a reverse proxy for your application server, and you can access it from a local browser if your application server is running. If you have an intended application server but do not have it running, you can proceed to starting your intended application server. You can skip the remainder of this tutorial.
Otherwise, proceed to setting up a test application and server with Gunicorn in the next step.
#### Step 3 — Testing your Reverse Proxy with Gunicorn (Optional)
If you had an application server prepared and running before beginning this tutorial, you can visit it in your browser now:
your_domain
However, if you don’t have an application server on hand to test your reverse proxy, you can go through the following steps to install Gunicorn along with a test application. Gunicorn is a Python WSGI server that is often paired with an Nginx reverse proxy.
Update your apt repository index and install gunicorn:
```
sudo apt update
sudo apt install gunicorn
```
You also have the option to install Gunicorn through pip with PyPI for the latest version that can be paired with a Python virtual environment, but apt is used here as a quick test bed.
Next, you’ll write a Python function to return “Hello World!” as an HTTP response that will render in a web browser. Create test.py using nano or your preferred text editor:
```
nano test.py
```
Insert the following Python code into the file:
```
def app(environ, start_response):
    start_response("200 OK", [])
    return iter([b"Hello, World!"])
```
This is the minimum required code by Gunicorn to start an HTTP response that renders a string of text in your web browser. After reviewing the code, save and close your file.
Now start your Gunicorn server, specifying the test Python module and the app function within it. Starting the server will take over your terminal:
```
gunicorn --workers=2 test:app
```
```
Output
[2022-08-29 07:09:29 +0000] [10568] [INFO] Starting gunicorn 20.1.0
[2022-08-29 07:09:29 +0000] [10568] [INFO] Listening at: http://127.0.0.1:8000 (10568)
[2022-08-29 07:09:29 +0000] [10568] [INFO] Using worker: sync
[2022-08-29 07:09:29 +0000] [10569] [INFO] Booting worker with pid: 10569
[2022-08-29 07:09:29 +0000] [10570] [INFO] Booting worker with pid: 10570
```
The output confirms that Gunicorn is listening at the default address of http://127.0.0.1:8000. This is the address that you set up previously in your Nginx configuration to proxy. If not, go back to your /etc/nginx/sites-available/your_domain file and edit the app_server_address associated with the proxy_pass directive.
Open your web browser and navigate to the domain you set up with Nginx:
```
your_domain
```
Your Nginx reverse proxy is now serving your Gunicorn web application server, displaying “Hello World!”.
### Conclusion
With this tutorial you have configured Nginx as a reverse proxy to enable access to your application servers that would otherwise only be available locally. Additionally, you configured the forwarding of request headers, passing on the client’s header information.
For examples of a complete solution using Nginx as a reverse proxy, check out how to serve Flask applications with Gunicorn and Nginx on Ubuntu 22.04 or how to run a Meilisearch frontend using InstantSearch on Ubuntu 22.04.

## How To Install and Use PostgreSQL on Ubuntu 22.04
### Introduction
Relational database management systems are a key component of many web sites and applications. They provide a structured way to store, organize, and access information.
PostgreSQL, or Postgres, is a relational database management system that provides an implementation of the SQL querying language. It’s standards-compliant and has many advanced features like reliable transactions and concurrency without read locks.

This guide demonstrates how to install Postgres on an Ubuntu 22.04 server. It also provides some instructions for general database administration.
#### Prerequisites
To follow along with this tutorial, you will need one Ubuntu 22.04 server that has been configured by following our Initial Server Setup for Ubuntu 22.04 guide. After completing this prerequisite tutorial, your server should have a non-root user with sudo permissions and a basic firewall.
#### Step 1 — Installing PostgreSQL
Ubuntu’s default repositories contain Postgres packages, so you can install these using the apt packaging system.
If you’ve not done so recently, refresh your server’s local package index:
```
sudo apt update
```
Then, install the Postgres package along with a -contrib package that adds some extra utilities and functionality:
```
sudo apt install postgresql postgresql-contrib
```
Ensure that the server is running using the systemctl start command:
```
sudo systemctl start postgresql.service
```
Now that the software is installed and running, we can go over how it works and how it may be different from other relational database management systems you may have used.
#### Step 2 — Using PostgreSQL Roles and Databases
By default, Postgres uses a concept called roles to handle authentication and authorization. These are, in some ways, similar to regular Unix-style accounts, but Postgres does not distinguish between users and groups and instead prefers the more flexible term “role”.
Upon installation, Postgres is set up to use peer authentication, meaning that it associates Postgres roles with a matching Unix/Linux system account. If a role exists within Postgres, a Unix/Linux username with the same name is able to sign in as that role.
The installation procedure created a user account called postgres that is associated with the default Postgres role. In order to use Postgres, you can log into that account.
There are a few ways to use this account to access Postgres.
Switching Over to the postgres Account
Switch over to the postgres account on your server by typing:
```
sudo -i -u postgres
```
You can now access the PostgreSQL prompt immediately by typing:
```
psql
```
From there you are free to interact with the database management system as necessary.
Exit out of the PostgreSQL prompt by typing:
```
\q
```
This will bring you back to the postgres Linux user’s command prompt.
Accessing a Postgres Prompt Without Switching Accounts
You can also run the command you’d like with the postgres account directly with sudo.
For instance, in the last example, you were instructed to get to the Postgres prompt by first switching to the postgres user and then running psql to open the Postgres prompt. You could do this in one step by running the single command psql as the postgres user with sudo, like this:
```
sudo -u postgres psql
```
This will log you directly into Postgres without the intermediary bash shell in between.
Again, you can exit the interactive Postgres session by typing:
```
\q
```
Many use cases require more than one Postgres role. Read on to learn how to configure these.
#### Step 3 — Creating a New Role
Currently, you have the postgres role configured within the database. You can create new roles from the command line with the createuser command. The --interactive flag will prompt you for the name of the new role and also ask whether it should have superuser permissions.
If you are logged in as the postgres account, you can create a new user by typing:
```
createuser --interactive
```

If, instead, you prefer to use sudo for each command without switching from your normal account, type:
```
sudo -u postgres createuser --interactive
```
The script will prompt you with some choices and, based on your responses, execute the correct Postgres commands to create a user to your specifications.
```
Output
Enter name of role to add: sammy
Shall the new role be a superuser? (y/n) y
```
You can get more control by passing some additional flags. Check out the options by looking at the man page for the createuser command:
```
man createuser
```
Your installation of Postgres now has a new user, but you have not yet added any databases. The next section describes this process.
#### Step 4 — Creating a New Database
Another assumption that the Postgres authentication system makes by default is that for any role used to log in, that role will have a database with the same name which it can access.
This means that if the user you created in the last section is called sammy, that role will attempt to connect to a database also called “sammy” by default. You can create the appropriate database with the createdb command.
If you are logged in as the postgres account, you would type something like:
```
createdb sammy
```
If, instead, you prefer to use sudo for each command without switching from your normal account, you would type:
```
sudo -u postgres createdb sammy
```

This flexibility provides multiple paths for creating databases as needed.
#### Step 5 — Opening a Postgres Prompt with the New Role
To log in with peer authentication, you’ll need a Linux user with the same name as your Postgres role and database.
If you don’t have a matching Linux user available, you can create one with the adduser command. You will have to do this from your non-root account with sudo privileges (meaning, not logged in as the postgres user):
```
sudo adduser sammy
```
Once this new account is available, you can either switch over and connect to the database by typing:
```
sudo -i -u sammy
psql
```
Or, you can do this inline:
```
sudo -u sammy psql
```
This command will log you in automatically, assuming that all of the components have been properly configured.
If you want your user to connect to a different database, you can do so by specifying the database like this:
```
psql -d postgres
```
Once logged in, you can get check your current connection information by typing:
```
\conninfo
```
Output
```
You are connected to database "sammy" as user "sammy" via socket in "/var/run/postgresql" at port "5432".
```
This is useful if you are connecting to non-default databases or with non-default users.
#### Step 6 — Creating and Deleting Tables
Now that you know how to connect to the PostgreSQL database system, you can learn some basic Postgres management tasks.
The basic syntax for creating tables is as follows:
```
CREATE TABLE table_name (
    column_name1 col_type (field_length) column_constraints,
    column_name2 col_type (field_length),
    column_name3 col_type (field_length)
);
```
This command gives the table a name, and then defines the columns as well as the column type and the max length of the field data. Optionally, you can add constraints for each column.
You can learn more about creating tables by following our guide on How To Create and Manage Tables in SQL.
For demonstration purposes, create the following table:
```
CREATE TABLE playground (
   equip_id serial PRIMARY KEY,
   type varchar (50) NOT NULL,
   color varchar (25) NOT NULL,
   location varchar(25) check (location in ('north', 'south', 'west', 'east', 'northeast', 'southeast', 'southwest', 'northwest')),
   install_date date
);
```
This command will create a table that inventories playground equipment. The first column in the table will hold equipment ID numbers of the serial type, which is an auto-incrementing integer. This column also has the constraint of PRIMARY KEY which means that the values within it must be unique and not null.
The next two lines create columns for the equipment type and color respectively, neither of which can be null. The line after these creates a location column with a constraint that requires the value to be one of eight possible values. The last line creates a date column that records the date on which you installed the equipment.
For two of the columns (equip_id and install_date), the command doesn’t specify a field length. The reason for this is that some data types don’t require a set length because the length or format is implied.
Examine your new table by typing:
```
\d
```
##### List of relations                                
| Schema |          Name           |   Type   | Owner |
|--------|-------------------------|----------|-------|
| public | playground              | table    | sammy |
| public | playground_equip_id_seq | sequence | sammy |

Your playground table is here, but there’s also something called playground_equip_id_seq that is of the type sequence. This is a representation of the serial type which you gave your equip_id column. This keeps track of the next number in the sequence and is created automatically for columns of this type.
If you want to view just the table without the sequence, you can type:
```
\dt
```

##### List of relations

| Schema |    Name    | Type  | Owner |
|--------|------------|-------|-------|
| public | playground | table | sammy |

With a table at the ready, let’s use it to practice managing data.
#### Step 7 — Adding, Querying, and Deleting Data in a Table
Now that you have a table, you can insert some data into it. As an example, add a slide and a swing by calling the table you want to add to, naming the columns and then providing data for each column, like this:
```
INSERT INTO playground (type, color, location, install_date) VALUES ('slide', 'blue', 'south', '2017-04-28');
INSERT INTO playground (type, color, location, install_date) VALUES ('swing', 'yellow', 'northwest', '2018-08-16');
```
You should take care when entering the data to avoid a few common hangups. For one, do not wrap the column names in quotation marks, but the column values that you enter do need quotes.
Another thing to keep in mind is that you do not enter a value for the equip_id column. This is because this is automatically generated whenever you add a new row to the table.
Retrieve the information you’ve added by typing:
```
SELECT * FROM playground;
```

Output
equip_id | type  | color  | location  | install_date 
----------+-------+--------+-----------+--------------
        1 | slide | blue   | south     | 2017-04-28
        2 | swing | yellow | northwest | 2018-08-16
(2 rows)
Notice that your equip_id has been filled in successfully and that all of your other data has been organized correctly.
If the slide on the playground breaks and you have to remove it, you can also remove the row from your table by typing:
```
DELETE FROM playground WHERE type = 'slide';
```

Query the table again:
```
SELECT * FROM playground;
```

Output
equip_id | type  | color  | location  | install_date 
----------+-------+--------+-----------+--------------
        2 | swing | yellow | northwest | 2018-08-16
(1 row)

Notice that the slide row is no longer a part of the table.
#### Step 8 — Adding and Deleting Columns from a Table
After creating a table, you can modify it by adding or removing columns. Add a column to show the last maintenance visit for each piece of equipment by typing:
```
ALTER TABLE playground ADD last_maint date;
```

View your table information again. A new column has been added but no data has been entered:
```
SELECT * FROM playground;
```

Output
equip_id | type  | color  | location  | install_date | last_maint 
----------+-------+--------+-----------+--------------+------------
        2 | swing | yellow | northwest | 2018-08-16   | 
(1 row)
If you find that your work crew uses a separate tool to keep track of maintenance history, you can delete of the column by typing:
```
ALTER TABLE playground DROP last_maint;
```
This deletes the last_maint column and any values found within it, but leaves all the other data intact.
#### Step 9 — Updating Data in a Table
So far, you’ve learned how to add records to a table and how to delete them, but this tutorial hasn’t yet covered how to modify existing entries.
You can update the values of an existing entry by querying for the record you want and setting the column to the value you wish to use. You can query for the swing record (this will match every swing in your table) and change its color to red. This could be useful if you gave the swing set a paint job:
```
UPDATE playground SET color = 'red' WHERE type = 'swing';
```
You can verify that the operation was successful by querying the data again:
```
SELECT * FROM playground;
```

Output
equip_id | type  | color | location  | install_date 
----------+-------+-------+-----------+--------------
        2 | swing | red   | northwest | 2018-08-16
(1 row)
The slide is now registered as being red.
### Conclusion
You are now set up with PostgreSQL on your Ubuntu 22.04 server. If you’d like to learn more about Postgres and how to use it, we encourage you to check out the following guides:
A comparison of relational database management systems
Practice running queries with SQL

## How To Install And Configure pgAdmin 4 in Server Mode on Ubuntu 22.04
### Introduction
pgAdmin is an open-source administration and development platform for PostgreSQL and its related database management systems. Written in Python and jQuery, it supports all the features found in PostgreSQL. You can use pgAdmin to do everything from writing basic SQL queries to monitoring your databases and configuring advanced database architectures.

In this tutorial, we’ll walk through the process of installing and configuring the latest version of pgAdmin onto an Ubuntu 22.04 server, accessing pgAdmin through a web browser, and connecting it to a PostgreSQL database on your server.
### Prerequisites
To complete this tutorial, you will need:

A server running Ubuntu 22.04. This server should have a non-root user with sudo privileges, as well as a firewall configured with ufw. For help with setting this up, follow our Initial Server Setup Guide for Ubuntu 22.04.
Nginx installed and configured as a reverse proxy for http://unix:/tmp/pgadmin4.sock, following Step 1 and 2 of How To Configure Nginx as a Reverse Proxy on Ubuntu 22.04.
PostgreSQL installed on your server. You can set this up by following our guide on How To Install and Use PostgreSQL on Ubuntu 22.04. As you follow this guide, be sure to create a new role and database, as you will need both to connect pgAdmin to your PostgreSQL instance.
Python 3 and venv installed on your server. Follow How To Install Python 3 and Set Up a Programming Environment on an Ubuntu 22.04 server to install these tools and set up a virtual environment.
#### Step 1 — Installing pgAdmin and its Dependencies
As of this writing, the most recent version of pgAdmin is pgAdmin 4, while the most recent version available through the official Ubuntu repositories is pgAdmin 3. pgAdmin 3 is no longer supported though, and the project maintainers recommend installing pgAdmin 4. In this step, we will go over the process of installing the latest version of pgAdmin 4 within a virtual environment (as recommended by the project’s development team) and installing its dependencies using apt.
To begin, update your server’s package index if you haven’t done so recently:
```
sudo apt update
```

Next, install the following dependencies. These include libgmp3-dev, a multiprecision arithmetic library; libpq-dev, which includes header files and a static library that helps communication with a PostgreSQL backend:
```
sudo apt install libgmp3-dev libpq-dev
```

Following this, create a few directories where pgAdmin will store its sessions data, storage data, and logs:
```
sudo mkdir -p /var/lib/pgadmin4/sessions
sudo mkdir /var/lib/pgadmin4/storage
sudo mkdir /var/log/pgadmin4
```

Then, change ownership of these directories to your non-root user and group. This is necessary because they are currently owned by your root user, but we will install pgAdmin from a virtual environment owned by your non-root user, and the installation process involves creating some files within these directories. After the installation, however, we will change the ownership over to the www-data user and group so it can be served to the web:
```
sudo chown -R sammy:sammy /var/lib/pgadmin4
sudo chown -R sammy:sammy /var/log/pgadmin4
```

Next, open up your virtual environment. Navigate to the directory your programming environment is in and activate it. Following the naming conventions of the prerequisite Python 3 tutorial, we’ll go to the environments directory and activate the my_env environment:
```
cd environments/
source my_env/bin/activate
```

After activating the virtual environment, it would be prudent to ensure that you have the latest version of pip installed on your system. To upgrade pip to the latest version, run the following command:
```
python -m pip install -U pip
Use pip to install pgadmin4:
python -m pip install pgadmin4==6.10
```

Next, install Gunicorn, a Python WSGI server that will be used with Nginx to serve the pgadmin web interface later in the tutorial:
```
python -m pip install gunicorn
```
That takes care of installing pgAdmin and its dependencies. Before connecting it to your database, though, there are a few changes you’ll need to make to the program’s configuration.
### Step 2 — Configuring pgAdmin 4
Although pgAdmin has been installed on your server, there are still a few steps you must go through to ensure it has the permissions and configurations needed to allow it to correctly serve the web interface.
pgAdmin’s main configuration file, config.py, is read before any other configuration file. Its contents can be used as a reference point for further configuration settings that can be specified in pgAdmin’s other config files, but to avoid unforeseen errors, you should not edit the config.py file itself. We will add some configuration changes to a new file, named config_local.py, which will be read after the primary one.
Create this file now using your preferred text editor. Here, we will use nano:
```
nano my_env/lib/python3.10/site-packages/pgadmin4/config_local.py
```
In your editor, add the following content:
```bash
environments/my_env/lib/python3.10/site-packages/pgadmin4/config_local.py
LOG_FILE = '/var/log/pgadmin4/pgadmin4.log'
SQLITE_PATH = '/var/lib/pgadmin4/pgadmin4.db'
SESSION_DB_PATH = '/var/lib/pgadmin4/sessions'
STORAGE_DIR = '/var/lib/pgadmin4/storage'
SERVER_MODE = True
```
Here are what these five directives do:
```ruby
LOG_FILE: this defines the file in which pgAdmin’s logs will be stored.
SQLITE_PATH: pgAdmin stores user-related data in an SQLite database, and this directive points the pgAdmin software to this configuration database. Because this file is located under the persistent directory /var/lib/pgadmin4/, your user data will not be lost after you upgrade.
SESSION_DB_PATH: specifies which directory will be used to store session data.
STORAGE_DIR: defines where pgAdmin will store other data, like backups and security certificates.
SERVER_MODE: setting this directive to True tells pgAdmin to run in Server mode, as opposed to Desktop mode.
```
Notice that each of these file paths point to the directories you created in Step 1.
After adding these lines, save and close the file. If you used nano, do so by press CTRL + X followed by Y and then ENTER.
With those configurations in place, run the pgAdmin setup script to set your login credentials:
python my_env/lib/python3.10/site-packages/pgadmin4/setup.py


After running this command, you will see a prompt asking for your email address and a password. These will serve as your login credentials when you access pgAdmin later on, so be sure to remember or take note of what you enter here:
Output
. . .
Enter the email address and password to use for the initial pgAdmin user account:

Email address: sammy@example.com
Password:
Retype password:
With that, pgAdmin is fully configured. However, the program isn’t yet being served from your server, so it remains inaccessible. To resolve this, we will configure Gunicorn and Nginx to serve pgAdmin so you can access its user interface through a web browser.
Step 3 — Starting Gunicorn and Configuring Nginx
You will be using Gunicorn to serve pgAdmin as a web application. However, as an application server Gunicorn will only be available locally, and not accessible through the internet. To make it available remotely, you will need to use Nginx as a reverse proxy.
Having completed the prerequisite to set up Nginx as a reverse proxy, your Nginx configuration file will contain this:
/etc/nginx/sites-available/your_domain


server {
    listen 80;
    listen [::]:80;

    server_name your_domain www.your_domain;
        
    location / {
        proxy_pass http://unix:/tmp/pgadmin4.sock;
        include proxy_params;
    }
}
This reverse proxy configuration enables your Gunicorn server to be accessible in your local browser. Start your Gunicorn server with the pgAdmin application:
gunicorn --bind unix:/tmp/pgadmin4.sock --workers=1 --threads=25 --chdir ~/environments/my_env/lib/python3.10/site-packages/pgadmin4 pgAdmin4:app


Output
[2022-08-29 00:19:11 +0000] [7134] [INFO] Starting gunicorn 20.1.0
[2022-08-29 00:19:11 +0000] [7134] [INFO] Listening at: unix:/tmp/pgadmin4.sock (7134)
[2022-08-29 00:19:11 +0000] [7134] [INFO] Using worker: gthread
[2022-08-29 00:19:11 +0000] [7135] [INFO] Booting worker with pid: 7135
Note: Invoking Gunicorn in this manner ties the process to your terminal. For a more long-term solution, invoke Gunicorn with a program like Supervisor. You can follow this tutorial on how to install and manage Supervisor on Ubuntu and Debian VPS.
With Gunicorn acting as an application server made accessible by your Nginx reverse proxy, you are ready to access pgAdmin in your web browser.
Step 4 — Accessing pgAdmin
On your local machine, open up your preferred web browser and navigate to your server’s IP address:
http://your_domain
Once there, you’ll be presented with a login screen similar to the following:

Enter the login credentials you defined in Step 2, and you’ll be taken to the pgAdmin Welcome Screen:

Now that you’ve confirmed you can access the pgAdmin interface, all that’s left to do is to connect pgAdmin to your PostgreSQL database. Before doing so, though, you’ll need to make one minor change to your PostgreSQL superuser’s configuration.
Step 5 — Configuring your PostgreSQL User
If you followed the prerequisite PostgreSQL tutorial, you should already have PostgreSQL installed on your server with a new superuser role and database set up.
Next, go back to the pgAdmin 4 interface in your browser, and locate the Browser menu on the left hand side. Right-click on Servers to open a context menu, hover your mouse over Create, and click Server….

This will cause a window to pop up in your browser in which you’ll enter info about your server, role, and database.
In the General tab, enter the name for this server. This can be anything you’d like, but you may find it helpful to make it something descriptive. In our example, the server is named Sammy-server-1.

Next, click on the Connection tab. Because pgAdmin is running on the same machine as your Postgres database, you can connect using a Unix domain socket. Relative to TCP socket connections, Unix domain sockets are much more performant with lower latency. This method also skips the need to set up a password. In the Host name/address field, enter /var/run/postgresql. The Port should be set to 5432 by default, which will work for this setup, as that’s the default port used by PostgreSQL.
In the Maintenance database field, enter the name of the database you’d like to connect to. Note that this database must already be created on your server. Then, enter the PostgreSQL username you configured previously. Here, our created database is sammy and username is sammy.


























The empty fields in the other tabs are optional, and it’s only necessary that you fill them in if you have a specific setup in mind in which they’re required. Click the Save button, and the database will appear under the Servers in the Browser menu.
You’ve successfully connected pgAdmin4 to your PostgreSQL database. You can do just about anything from the pgAdmin dashboard that you would from the PostgreSQL prompt. To illustrate this, we will create an example table and populate it with some sample data through the web interface.
Step 6 — Creating a Table in the pgAdmin Dashboard
From the pgAdmin dashboard, locate the Browser menu on the left-hand side of the window. Click on the plus sign (+) next to Servers (1) to expand the tree menu within it. Next, click the plus sign to the left of the server you added in the previous step (Sammy-server-1 in our example), then expand Databases, the name of the database you added (sammy, in our example), and then Schemas (1). You should see a tree menu like the following:
Right-click the Tables list item, then hover your cursor over Create and click Table….






























This will open up a Create-Table window. Under the General tab of this window, enter a name for the table. This can be anything you’d like, but to keep things simple we’ll refer to it as table-01.
Then navigate to the Columns tab and click the + sign in the upper right corner of the window to add some columns. When adding a column, you’re required to give it a Name and a Data type, and you may need to choose a Length if it’s required by the data type you’ve selected.
Additionally, the official PostgreSQL documentation states that adding a primary key to a table is usually best practice. A primary key is a constraint that indicates a specific column or set of columns that can be used as a special identifier for rows in the table. This isn’t a requirement, but if you’d like to set one or more of your columns as the primary key, toggle the switch at the far right from No to Yes.
Click the Save button to create the table.


By this point, you’ve created a table and added a couple columns to it. However, the columns don’t yet contain any data. To add data to your new table, right-click the name of the table in the Browser menu, hover your cursor over Scripts and click on INSERT Script.


This will open a new panel on the dashboard. At the top you’ll see a partially-completed INSERT statement, with the appropriate table and column names. Go ahead and replace the question marks (?) with some dummy data, being sure that the data you add aligns with the data types you selected for each column. Note that you can also add multiple rows of data by adding each row in a new set of parentheses, with each set of parentheses separated by a comma as shown in the following example.
If you’d like, feel free to replace the partially-completed INSERT script with this example INSERT statement:
INSERT INTO public."table-01"(
    col1, col2, col3)
    VALUES ('Juneau', 14, 337), ('Bismark', 90, 2334), ('Lansing', 51, 556);


Click on the sideways triangle icon (▶) to execute the INSERT statement. Note that in older versions of pgAdmin, the execute icon is instead a lightning bolt (⚡).
To view the table and all the data within it, right-click the name of your table in the Browser menu once again, hover your cursor over View/Edit Data, and select All Rows.

This will open another new panel, below which, in the lower panel’s Data Output tab, you can view all the data held within that table.

With that, you’ve successfully created a table and populated it with some data through the pgAdmin web interface. Of course, this is just one method you can use to create a table through pgAdmin. For example, it’s possible to create and populate a table using SQL instead of the GUI-based method described in this step.
Conclusion
In this guide, you learned how to install pgAdmin 4 from a Python virtual environment, configure it, serve it to the web with Gunicorn and Nginx, and how to connect it to a PostgreSQL database. Additionally, this guide went over one method that can be used to create and populate a table, but pgAdmin can be used for much more than just creating and editing tables.
For more information on how to get the most out of all of pgAdmin’s features, we encourage you to review the project’s documentation. You can also learn more about PostgreSQL through our Community tutorials on the subject.
website log in : http://143.198.81.111/browser/

username : afiqazwan25@gmail.com
password : Dyna1234

Those code is to run the pgadmin website in server :

cd environments/
source my_env/bin/activate

gunicorn --bind 127.0.0.1 --workers=1 --threads=25 --chdir ~/environments/my_env/lib/python3.12/site-packages/pgadmin4 pgAdmin4:app
