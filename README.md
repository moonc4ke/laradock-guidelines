# Laradock Guidelines

## Step 1: Install Laradock

Laradock is the docker version of Laravel homestead. Unlike homestead, it is highly scalable and production-ready.
Go to https://laradock.io/ and check out their wonderful quick start guide.
Git clone of https://github.com/laradock/laradock and place it in your Project folder. Folder structure should look like this:

```bash
+ laradock
+ project-1
+ project-2
```

## Step 2: Configure Laradock

1. Set up sites
    1. Use Nginx.
    2. In laradock > nginx > sites, make a copy of the laravel.conf.example and rename it to your-site.conf
    3. Change the server name and the root folder location.
    4. Uncomment the https block if you would like to use the https protocol.
    5. Should have something like this in the end:
    
![your-site.conf](/images/conf.png)

2. Configure laradock .env file
    1. Rename COMPOSE_PROJECT_NAME to define the prefix of container names. For example, "bulve" or "patatas"
    2. Under php-fpm:
       * Set PHP_FPM_INSTALL_XDEBUG=true
    3. Under MySQL
       * Change MYSQL_VERSION from “latest” to 5.7.25. This is a workaround for a bug in MySQL v8
       * Change MYSQL_PORT if you have a local MySQL server installed already
    4. Under workspace
       * Set WORKSPACE_INSTALL_PYTHON=true
       * Set WORKSPACE_INSTALL_WORKSPACE_SSH=true
    5. Under PHP MY ADMIN
        * Change PMA_PORT to 8888
       
3. Configure docker-compose.yml
    1. Under workspace > ports
        * Add 8080:8080 — this is for vue hot reloading
    2. Under workspace > extra_hosts:
        * Add “larvel-test.dev.local:0.0.0.0” — this is to make sure vue hot reload server can resolve the host machine’s IP
    
4. Configure xdebug.ini in php-fpm:

![xdebug.ini](/images/xdebuginic.png)

5. Increase the Session Timeout for phpMyAdmin:
    1. Open docker-compose.yml and add volume for phpMyAdmin:
    
    ![phpMyAdmin volume](/images/phpmyadmin_volume.png)
    
    2. Create [config.inc.php](https://github.com/s-emicolon/laradock-guidelines/blob/master/config.inc.php) in phpmyadmin directory:
    
    ![phpMyAdmin config dir](/images/phpmyadmin_config.png)

6. Enable port 9000 for XDebug to work in yout Ubuntu machine by running:

```bash
$ sudo ufw allow 9000
```

## Step 3: Add your development site to your OS’s host file

```bash
$ sudo nano /etc/hosts
```

In the host file you should add an entry that looks something like this:

```bash
127.0.0.1       laravel-test.dev.local
```

## Step 4: Create a New Laravel Project & Configure PHPStorm

1. Build the images and start the containers before continuing. Run the following command in the laradock folder under your project directory:

```bash
$ docker-compose build nginx php-fpm mysql phpmyadmin

# nginx is dependent on php-fpm, starting nginx will automatically 
# start php-fpm
$ docker-compose up -d nginx mysql phpmyadmin

# to view all started container
$ docker-compose ps

# you can ssh into your workspace container by running:
$ docker-compose exec --user=laradock workspace bash
```
2. Once you ssh into your workspace container, you can create a new Laravel project by running:

```bash
$ composer create-project --prefer-dist laravel/laravel your-project-name
```
* After creating new Laravel project make sure you are using correct database credentials in .env file:

![db credentials](/images/db_credentials.png)

3. Add PHP CLI Interpreter & Path mappings in PhpStorm:

![CLI Interpreter 1](/images/cli_interpreter_1.png)

![CLI Interpreter 2](/images/cli_interpreter_2.png)

![CLI Interpreter 3](/images/cli_interpreter_3.png)

![CLI Interpreter 4](/images/cli_interpreter_4.png)

![Path mappings 1](/images/path_mappings_1.png)

![Path mappings 2](/images/path_mappings_2.png)

![Path mappings 3](/images/path_mappings_3.png)

4. Add a new server named ‘laradock’ in PhpStorm:

![phpstorm server](/images/phpstormserver.png)

5. Make sure yout PhpStorm PHP > Debug settings look like this:

![phpstorm debug](/images/debug.png)

6. Alt + Shift + F9 in PhpStorm and then press 0 to create PHP Remote Debug configuration:

![php remote debug configuration](/images/phpremotedebug.png)

7. Create /xdebug-test route in web.php Laravel routes to test if XDebug is working:

```bash
Route::get('/xdebug-test', function () {
    $word = 'Debugger';

    $word .= ' In Action';

    echo $word;
});
```

Once, created F9 and then 1 to start debugger. Now you should be able to debug:

![phpstorm xdebug](/images/phpstorm_xdebug.png)

8. To run **PHPUnit** test simply open test file and press F9. You should see PHPUnit test option in the menu - select that option and your PHPUnit test should run flawlessly 

9. Set up Laravel Telescope https://laravel.com/docs/6.x/telescope

## Step 5: Set up Vue Hot Reloading HMR

1. Add the following code in Laravel project's webpack.mix.js:

```javascript
mix.options({
    hmrOptions: {
        host: 'laravel.dev.local',  // site's host name
        port: 8080,
    }
});

// // fix css files 404 issue
mix.webpackConfig({
    // add any webpack dev server config here
    devServer: { 
        proxy: {
            host: '0.0.0.0',  // host machine ip 
            port: 8080,
        },
        watchOptions:{
            aggregateTimeout:200,
            poll:5000
        },

    }
});
```

2. Add/ change modified app.js and app.css includes in Laravel blade files:

```bash
<link rel="stylesheet" href="{{ mix('/css/app.css') }}">

<script src="{{ mix('js/app.js') }}"></script>
```


3. If you want to run your development site on https, use the following ‘hot’ option in Laravel project's package.json:

```bash
{
....

"hot": "cross-env NODE_ENV=development node_modules/webpack-dev-server/bin/webpack-dev-server.js --inline --hot --https --key /etc/ssl/private/default.key --cert /etc/ssl/private/default.crt --config=node_modules/laravel-mix/setup/webpack.config.js",
...
}
```

4. The hot reload server runs in laradock workspace container. Make sure you have set up the SSL key and cert correctly. Here’s a script for generating the SSL in my workspace container:

```bash
# at the end of the workspace dockerfile

RUN openssl dhparam -out /etc/ssl/private/dhparam.pem 2048

RUN openssl genrsa -out "/etc/ssl/private/default.key" 2048

RUN openssl req -new -key "/etc/ssl/private/default.key" -out "/etc/ssl/private/default.csr" -subj "/CN=default/O=default/C=UK"

RUN openssl x509 -req -days 3650 -in "/etc/ssl/private/default.csr" -signkey "/etc/ssl/private/default.key" -out "/etc/ssl/private/default.crt"
```
## Step 6: Complete phpDocs, directly from the source with ide-helper command:

In your laradock folder, run:

```bash
# start the containers
docker-compose up -d nginx mysql phpmyadmin

# ssh into the workspace container
docker-compose exec --user=laradock workspace bash

# require ide-helper package with composer using the following command
composer require --dev barryvdh/laravel-ide-helper

# complete phpDocs
php artisan ide-helper:models -n && php artisan ide-helper:eloquent && php artisan ide-helper:generate && php artisan ide-helper:meta
```

To fix "validate method not found in Illuminate\Http\Request" - create IDEAutoCompleteHelp.php at project's root folder:

```bash
<?php

namespace Illuminate\Http;

/**
 * @method bool|array|null validate(array $rules, ...$params) Validate the given request with the given rules.
 * @method array           validated()                        Get the validated data from the request.
 */
class Request {

}
```

If you experience "Undefined class object - multiple definitions exist for class", do this:

```bash
Settings | Editor | Inspections | PHP | Undefined | Undefined class | Don't report multiple class declaration potential problems
```

To fix navigating from route to controller - change _ide_helper.php:

```bash
class Route extends \Illuminate\Support\Facades\Route {}
```

to

```bash
class Route extends Illuminate\Support\Facades\Route {}
```

To remove "property accessed via magic method" without document comments you can uncheck **Notify about access to a field via magic method** which is found in:

```bash
Project Settings > Inspections > PHP > Undefined > Undefined property
```


## Step 7: Modify Laravel project's .gitignore

```bash
/.idea
/node_modules
/public/hot
/public/storage
/storage/*.key
/vendor
.env
.env.backup
.phpunit.result.cache
Homestead.json
Homestead.yaml
npm-debug.log
yarn-error.log
.phpstorm.meta.php
_ide_helper.php
_ide_helper_models.php
```

## Step 8: PHP Unit Tests - "ReflectionException: Class env does not exist" error (ALWAYS USE DIFFERENT TESTING DATABASE FOR A PROJECT TO AVOID TELESCOPE FROM CRASHING):

#### Solution:

* Copy .env to .env.testing
* Open .env.testing
* Change TELESCOPE_ENABLED=true to TELESCOPE_ENABLED=false

#### If Testing With Dusk:

* Copy .env to .env.testing
* Open .env.testing
* Change TELESCOPE_ENABLED=true to TELESCOPE_ENABLED=false
* Copy .env.testing to .env.dusk.local

## Step 9: Start coding!

In your laradock folder, run:

```bash
# start the containers
docker-compose up -d nginx mysql phpmyadmin

# ssh into the workspace container
docker-compose exec --user=laradock workspace bash

# start hot reloading server
cd my-project/ 
yarn hot
```

## How to access phpMyAdmin

Make sure that containers are running, if not start the containers by running:

```bash
docker-compose up -d nginx mysql phpmyadmin
```

Then, you can open phpMyAdmin by accessing http://localhost:8888/

![phpMyAdmin](/images/phpmyadmin.png)

## Production Setup
### Run Site on SSL with Let’s Encrypt Certificate

**Note: You need to Use Caddy here Instead of Nginx**

To go Caddy Folders and Edit CaddyFile

```bash
$root@server:~/laravel/laradock# cd caddy
$root@server:~/laravel/laradock/caddy# nano Caddyfile
```

Remove 0.0.0.0:80

```bash
0.0.0.0:80
root /var/www/public
```

and replace with your https://yourdomain.com

```bash
https://yourdomain.com
root /var/www/public
```

uncomment tls

```bash
#tls self-signed
```

and replace self-signed with your email address

```bash
tls serverbreaker@gmai.com
```

This is needed Prior to Creating Let’s Encypt

### Run Your Caddy Container without the -d flag and Generate SSL with Let’s Encrypt

```bash
$root@server:~/laravel/laradock# docker-compose up  caddy
```

You’ll be prompt here to enter your email… you may enter it or not

```bash
Attaching to laradock_mysql_1, laradock_caddy_1
caddy_1               | Activating privacy features...
caddy_1               | Your sites will be served over HTTPS automatically using Let's Encrypt.
caddy_1               | By continuing, you agree to the Let's Encrypt Subscriber Agreement at:
caddy_1               |   https://letsencrypt.org/documents/LE-SA-v1.0.1-July-27-2015.pdf
caddy_1               | Activating privacy features... done.
caddy_1               | https://yourdomain.com
caddy_1               | http://yourdomain.com
```

After it finishes, press Ctrl + C to exit.

### Stop All Containers and ReRun Caddy and Other Containers on Background

```bash
$root@server:~/laravel/laradock# docker-compose down
$root@server:~/laravel/laradock# docker-compose up -d mysql caddy
```

View your Site in the Browser Securely Using HTTPS (https://yourdomain.com)

#### Note that Certificate will be Automatically Renew By Caddy
