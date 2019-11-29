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
    4. Uncomment the https block if you would like to use the https protocol (recommended).
    5. Should have something like this in the end:
    
![your-site.conf](/images/conf.png)

2. Configure laradock .env file
    1. Rename COMPOSE_PROJECT_NAME to your project name.
    2. Under php-fpm:
       * Set PHP_FPM_INSTALL_XDEBUG=true
    3. Under MySQL
       * Change MYSQL_VERSION from “latest” to 5.7.25. This is a workaround for a bug in MySQL v8
       * Change MYSQL_PORT if you have a local MySQL server installed already
    4. Under workspace
       * Set WORKSPACE_INSTALL_PYTHON=true
       * Set WORKSPACE_INSTALL_WORKSPACE_SSH=true
       
3. Configure docker-compose.yml
    1. Under workspace > ports
        * Add 8080:8080 — this is for vue hot reloading
    2. Under workspace > extra_hosts:
        * Add “${COMPOSE_PROJECT_NAME}.dev.local:0.0.0.0” — this is to make sure vue hot reload server can resolve the host machine’s IP

4. Configure xdebug.ini in php-fpm:

![xdebug.ini](/images/xdebug.ini.png)

5. Enable port 9000 for XDebug to work in yout Ubuntu machine by running:

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
$ docker-compose build nginx php-fpm mysql 

# nginx is dependent on php-fpm, starting nginx will automatically 
# start php-fpm
$ docker-compose up -d nginx mysql

# to view all started container
$ docker-compose ps

# you can ssh into your workspace container by running:
$ docker-compose exec --user=laradock workspace bash
```
2. Once you ssh into your workspace container, you can create a new Laravel project by running:

```bash
$ composer create-project --prefer-dist laravel/laravel your-project-name
```
After creating new Laravel project make sure you are using correct database credentials in .env file:

![db credentials](/images/db_credentials.png)

3. Add a new server named ‘laradock’ in PhpStorm:

![phpstorm server](/images/phpstormserver.png)

4. Make sure yout PhpStorm PHP > Debug settings look like this:

![phpstorm debug](/images/debug.png)

5. Alt + Shift + F9 in PhpStorm and then press 0 to create PHP Remote Debug configuration:

![php remote debug configuration](/images/phpremotedebug.png)

6. Create /xdebug-test route in web.php Laravel routes to test if XDebug is working:

```bash
Route::get('/xdebug-test', function () {
    $word = 'Debugger';

    $word .= ' In Action';

    echo $word;
});
```

Once, created Alt + Shift + F9 and then 1 to start debugger. Now you should be able to debug:

![phpstorm xdebug](/images/phpstorm_xdebug.png)

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

2. If you want to run your development site on https, use the following ‘hot’ option in Laravel project's package.json:

```bash
{
....

"hot": "cross-env NODE_ENV=development node_modules/webpack-dev-server/bin/webpack-dev-server.js --inline --hot --https --key /etc/ssl/private/default.key --cert /etc/ssl/private/default.crt --config=node_modules/laravel-mix/setup/webpack.config.js",
...
}
```

3. The hot reload server runs in laradock workspace container. Make sure you have set up the SSL key and cert correctly. Here’s a script for generating the SSL in my workspace container:

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
docker-compose up -d nginx mysql


# ssh into the workspace container
docker-compose exec --user=laradock workspace bash

# require ide-helper package with composer using the following command
composer require --dev barryvdh/laravel-ide-helper

# complete phpDocs
php artisan ide-helper:models -n && php artisan ide-helper:eloquent && php artisan ide-helper:generate && php artisan ide-helper:meta
```

## Step 7: Start coding!

In your laradock folder, run:

```bash
# start the containers
docker-compose up -d nginx mysql

# ssh into the workspace container
docker-compose exec --user=laradock workspace bash

# start hot reloading server
cd my-project/ 
yarn hot
```
