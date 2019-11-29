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
![your-site.conf](http://via.placeholder.com/200x150)
