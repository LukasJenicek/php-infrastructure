## Docker local development

Todolist:
- Using alpine (checked)
- Installing and Running composer (checked)
- Install and configure xdebug (checked)
- Install bash instead of ash (checked)
- Install zsh (checked)

How to run and exec container
 - docker run -it -v $(pwd)/../project:/app --entrypoint ash php-cli:1.0.0
 - if you want more go to documentation but enought is running docker run --help and see what
   each parameter means
 - parameter "i" -> Interactive Keep STDIN open even if not attached -> interactively acts on the input you put there 
   without it the commands in new tty would not work  
 - parameter "t" -> pseudo tty 
 - never use -i without -t doesnt make sense basically
 - the combination -it ->  It basically makes the container start look like a terminal connection session.

Command tty ? 
 - prints name of your terminal you are using
 
 So we have a really simple php-cli container what next ? 
 - we would like to have a composer in it right ? we are developing a php application so without composer we are screwed
 - for us to be able download composer we will need the curl 
 - How to install curl in Dockerfile `RUN apt-get install curl`, but we got 
 `/bin/sh: apt-get: not found`. How is that possible ? We are using alpine based image and `apt-get` is Ubuntu package manager.
 - We could install ubuntu package manager but having more package managers is not a good idea so we just have to change 
 a command little bit
 `RUN apk update && apk add --virtual build-deps gcc python-dev musl-dev curl`
 - Then we finally can install composer via 
 `RUN curl -sS https://getcomposer.org/installer -o composer-setup.php && \
  php composer-setup.php --install-dir=/usr/local/bin --filename=composer`
 - We can enter the container and run composer --version it should display the latest version of composer. 
  It's still far away from ideal situation but we are getting there.
 - Not let's say we want to download any package from github let's try that and let's see what happens
 - This package will suffice `https://github.com/php-fig/container`, package should be downloaded but let's
   the permissions in open session of host run this command `ls -la` 
   the file `composer.json, composer.lock and vendor folder` are under group `root`. So
   we cannot modify them and it could be problem in the future we need somehow to connect our host user with the user in 
   container. 
   How we do that ? We have to add user inside container with same UID and GUID as our host user. 
   On a host container you can find out your user id and group id with this command `id -u / id -g` and this values we need to
   set to user inside of container. So the most common way is to create a new user with uid and gid of the user and 
   run the container as this newly created user.
   ```
   ARG UID=1000
   ARG GID=1000
   ARG USER=appuser
   
   RUN addgroup -S -g $UID $USER && adduser -u $GID -S $USER -G $USER 
   // then we can pass GID and UID parameter while building the container like this
   docker build . -t 'php-cli:1.0.0' --build-arg UID=$(id -u) --build-arg GID=$(id -g)
   ```
   
   Then set default directory whenever you enter the container like this 
   `WORKDIR app/` and we are almost set.
    
    Last step is get xdebug to work. Installation can be done like this `RUN pecl install xdebug && docker-php-ext-enable xdebug`
    but before we need to `phpize` be working properly and to do that just add `$PHPIZE_DEPS` to installation of packages.
    If you want to check what $PHPIZE_DEPS contains just enter the container run `printenv` and you will see the variable.
    Then install the xdebug extension. We need also config for the xdebug. So you create xdebug.ini file and 
    add this configuration.
    ```
   [xdebug]
   xdebug.max_nesting_level = 256
   
   xdebug.remote_enable       = 1
   xdebug.remote_autostart    = 1
   xdebug.remote_mode         = req
   xdebug.remote_connect_back = 1
   xdebug.remote_host         = 172.17.0.1
   xdebug.idekey              = "PHPSTORM"
   
   xdebug.profiler_append         = 0
   xdebug.profiler_enable         = 0
   xdebug.profiler_enable_trigger = 1
   xdebug.profiler_output_name    = cachegrind.out.%S
   ```
   
   Then to Dockerfile add this line `COPY config/* $PHP_INI_DIR/conf.d/` assuming you add the configuration 
   to folder config on your host.
   Only thing i can point out in the xdebug configuration is the remote_host value `172.17.0.1` which is equal to 
   docker0 interface. If you enter `ifconfig` in your terminal and there has to be definitely interface with name `docker0`. 
   You can read more about it here https://developer.ibm.com/recipes/tutorials/networking-your-docker-containers-using-docker0-bridge/.
   
   Then build image again and enter the container. Enter this command `export PHP_IDE_CONFIG="serverName=application"` 
   and then you have to configure the PHPSTORM `File | Settings | Languages & Frameworks | PHP | Servers`.
   
   The result Dockerfile should look like this
   
   ```
   FROM php:7.3-alpine
   
   ARG UID=1000
   ARG GID=1000
   ARG USER=appuser
   
   RUN apk update && \
       apk add --virtual build-deps gcc python-dev musl-dev curl unzip $PHPIZE_DEPS
   
   RUN curl -sS https://getcomposer.org/installer -o composer-setup.php && \
       php composer-setup.php --install-dir=/usr/local/bin --filename=composer
   
   RUN addgroup -S -g $UID $USER && adduser -u $GID -S $USER -G $USER
   
   RUN pecl install xdebug && docker-php-ext-enable xdebug
   
   COPY config/* $PHP_INI_DIR/conf.d/
   
   USER $USER
   WORKDIR app/
   ```

Todolist:
 - nginx handle html file
 - communication php-fpm with nginx

Create nginx configuration
```
    FROM nginx:1.17.5-alpine
```

If you want to test it run `docker run -p 8080:80 nginx:1.0.0` then you can enter localhost:8080 in your browser 
and then you find the default nginx html page displayed. You exposed port 80 to port 8080 on your host.


Now we want to make virtual host which will point to project/public/index.html so we have to create configuration 
for nginx. In the container configuration is saved on the common path /etc/nginx/conf.d where you can 
also find the default configuration.

Create new folder in nginx folder called `sites` where we store configuration per project. Just for the demo 
create configuration file called project.conf and put there.

```
server {
    listen       80;
    server_name  project.localhost;

    location / {
        root   /project/public;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

Now we have to apply all the configurations. Just open the Dockerfile for nginx and put there
`COPY sites/* /etc/nginx/conf.d/`. Now rebuild image again and enter project.localhost in browser. Your browser should 
respond with site not found or something like that because you have to add new virtual host which will point project.localhost 
to localhost. Every time you enter `project.localhost` it will be pointed to `localhost`  For this to be working you have to add this line
to hosts file `127.0.0.1 project.localhost`. Then when you try to refresh the browser content of the index.html stored 
in path `/project/public/index.html` should be displayed.
