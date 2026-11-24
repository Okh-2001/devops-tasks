# LEMP Stack Implementation on AWS

A LEMP stack hosts dynamic websites using four tools:

- **L**: Linux (Ubuntu Server on AWS EC2)
- **E**: Nginx (the web server)
- **M**: MySQL (the database)
- **P**: PHP (the scripting language)

## Environment

| Component | Details |
|---|---|
| Cloud provider | AWS EC2, t3.micro, region eu-north-1 |
| Operating system | Ubuntu Server 26.04 LTS |
| Web server | Nginx 1.28 |
| Database | MySQL 8.4 |
| Scripting language | PHP 8.5 with PHP-FPM |

## Step 0: Launch the EC2 instance

I launched a t3.micro Ubuntu instance named `project-2`, with SSH and HTTP allowed in the security group, then connected over SSH.

![Instance launch settings](screenshots/AWS-1.png)

![Security group allowing SSH and HTTP](screenshots/AWS-2.png)

![Instance in the running state](screenshots/AWS-3.png)

![SSH into my ubuntu instance](screenshots/SSH.png)

## Step 1: Install Nginx

Nginx is the web server that answers browser requests. I installed it and checked that it was running.

```bash
sudo apt update
sudo apt install nginx
sudo systemctl status nginx
curl http://localhost:80
```

![Updating the package list](screenshots/Nginx-1.png)

![Installing Nginx](screenshots/Nginx-2.png)

![Nginx service running](screenshots/Nginx-3.png)

![Nginx welcome page from curl](screenshots/Nginx-4.png)

I then opened the server's public IP in a browser and saw the Nginx welcome page.

![Nginx welcome page in the browser](screenshots/Browser-1.png)

## Step 2: Install MySQL

MySQL stores the website's data. I installed it, set a root password, and ran the security script.

```bash
sudo apt install mysql-server
sudo mysql
sudo mysql_secure_installation
sudo mysql -p
```

Inside the MySQL console, I set the root password:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'your_password';
exit
```

![Installing MySQL](screenshots/Mysql-1.png)

![Setting the root password](screenshots/Mysql-2.png)

![Running the security script](screenshots/Mysql-3.png)

![Logging in with a password](screenshots/Mysql-4.png)

## Step 3: Install PHP

Nginx needs `php-fpm` to run PHP code, and `php-mysql` lets PHP talk to MySQL.

```bash
sudo apt install php-fpm php-mysql
```

![Installing PHP](screenshots/Php-1.png)

## Step 4: Configure Nginx to use PHP

I made a folder for the site, took ownership of it, wrote a server block, enabled it, and disabled the default site. I also added an `index.html` file to test it.

```bash
sudo mkdir /var/www/projectLEMP
sudo chown -R $USER:$USER /var/www/projectLEMP
sudo vim /etc/nginx/sites-available/projectLEMP
sudo ln -s /etc/nginx/sites-available/projectLEMP /etc/nginx/sites-enabled/
sudo nginx -t
sudo unlink /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
```

The server block:

```nginx
server {
    listen 80;
    server_name projectLEMP www.projectLEMP;
    root /var/www/projectLEMP;
    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.5-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

![Setting up the server block](screenshots/Config-1.png)

Opening the public IP showed my test message, so the new server block works.

![Test page from index.html](screenshots/Browser-2.png)

## Step 5: Test PHP with Nginx

I created `info.php` to check that Nginx can pass PHP files to PHP-FPM.

```php
<?php
phpinfo();
```

The page first showed a `502 Bad Gateway` error.

![502 Bad Gateway error](screenshots/Browser-3.png)

I fixed the socket name in the Nginx config and the PHP info page loaded.

![Fixing the PHP socket name](screenshots/Php-2.png)

![PHP info page](screenshots/Browser-4.png)

After testing, I deleted the file because it exposes server details.

```bash
sudo rm /var/www/projectLEMP/info.php
```

![Removing info.php](screenshots/Php-3.png)

## Step 6: Get data from MySQL with PHP

I created a database, a user with access to it, and a `todo_list` table with four items.

```sql
CREATE DATABASE `example_database`;
CREATE USER 'example_user'@'%' IDENTIFIED WITH caching_sha2_password BY 'your_password';
GRANT ALL ON example_database.* TO 'example_user'@'%';

CREATE TABLE example_database.todo_list (
    item_id INT AUTO_INCREMENT,
    content VARCHAR(255),
    PRIMARY KEY(item_id)
);


INSERT INTO example_database.todo_list (content) VALUES ("My first important item");
INSERT INTO example_database.todo_list (content) VALUES ("My second important item");
INSERT INTO example_database.todo_list (content) VALUES ("My third important item");
INSERT INTO example_database.todo_list (content) VALUES ("and this one more thing");
SELECT * FROM example_database.todo_list;
```

![Creating the database and user](screenshots/Db-1.png)

![Logging in as example_user](screenshots/Db-2.png)

![Table with four items](screenshots/Db-3.png)

Then I wrote `todo_list.php` in the web root. It connects to MySQL and prints the items as a list.

```php
<?php
$user = "example_user";
$password = "password";
$database = "example_database";
$table = "todo_list";

try {
    $db = new PDO("mysql:host=localhost;dbname=$database", $user, $password);
    echo "<h2>TODO</h2><ol>";
    foreach ($db->query("SELECT content FROM $table") as $row) {
        echo "<li>" . $row['content'] . "</li>";
    }
    echo "</ol>";
} catch (PDOException $e) {
    print "Error!: " . $e->getMessage() . "<br/>";
    die();
}
```

![Todo list shown in the browser](screenshots/Browser-5.png)

## Problems I Hit and How I Fixed Them

| Problem | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` on `info.php` | Nginx pointed to `php8.1-fpm.sock`, but the server runs PHP 8.5 | Checked `ls -l /run/php/` and changed `fastcgi_pass` to `php8.5-fpm.sock` |
| `Plugin 'mysql_native_password' is not loaded` | MySQL 8.4 turns this plugin off | Used `caching_sha2_password` instead |
| Browser showed my router page instead of the site | My home router intercepts port 80 | Tested from my phone on mobile data and with `curl` |

## What I Learnt From This Project

- The difference between Apache and Nginx, and why Nginx needs PHP-FPM
- How to match a config file to the real PHP version and socket on the server
- How to create a MySQL database and a user with limited access
- How to troubleshoot with `curl`, service status, and log checks