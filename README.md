# Docker PHP-FPM 8.2 & Nginx 1.24 on Alpine Linux

Example PHP-FPM 8.2 & Nginx 1.24 setup for Docker, build on [Alpine Linux](https://www.alpinelinux.org/).
The image is only +/- 25MB large.

## Usage

Start the Docker container:

    docker run -p 80:8080 andergrim/alpine-php-fpm-nginx

See the PHP info on http://localhost, or the static html page on http://localhost/test.html

Or mount your own code to be served by PHP-FPM & Nginx

    docker run -p 80:8080 -v ~/my-codebase:/var/www/html andergrim/alpine-php-fpm-nginx


## Adding additional daemons
You can add additional daemons (e.g. your own app) to the image by creating runit entries. You only have to write a small shell script which runs your daemon, and runit will keep it up and running for you, restarting it when it crashes, etc.

The shell script must be called `run`, must be executable, and is to be placed in the directory `/etc/service/<NAME>`.

Here's an example showing you how a memcached server runit entry can be made.

    #!/bin/sh
    ### In memcached.sh (make sure this file is chmod +x):
    # `chpst -u memcache` runs the given command as the user `memcache`.
    # If you omit that part, the command will be run as root.
    exec 2>&1 chpst -u memcache /usr/bin/memcached

    ### In Dockerfile:
    RUN mkdir /etc/service/memcached
    ADD memcached.sh /etc/service/memcached/run

Note that the shell script must run the daemon **without letting it daemonize/fork it**. Usually, daemons provide a command line flag or a config file option for that.


## Running scripts during container startup
You can set your own scripts during startup, just add your scripts in `/docker-entrypoint-init.d/`. The scripts are run in lexicographic order.

All scripts must exit correctly, e.g. with exit code 0. If any script exits with a non-zero exit code, the booting will fail.

The following example shows how you can add a startup script. This script simply logs the time of boot to the file /tmp/boottime.txt.

    #!/bin/sh
    ### In logtime.sh (make sure this file is chmod +x):
    date > /tmp/boottime.txt

    ### In Dockerfile:
    ADD logtime.sh /docker-entrypoint-init.d/logtime.sh


## Configuration
In [rootfs/etc/](rootfs/etc/) you'll find the default configuration files for Nginx, PHP and PHP-FPM.
If you want to extend or customize that you can do so by mounting a configuration file in the correct folder;

Nginx configuration:

    docker run -v "`pwd`/nginx-server.conf:/etc/nginx/conf.d/server.conf" andergrim/alpine-php-fpm-nginx

PHP configuration:

    docker run -v "`pwd`/php-setting.ini:/etc/php8/conf.d/settings.ini" andergrim/alpine-php-fpm-nginx

PHP-FPM configuration:

    docker run -v "`pwd`/php-fpm-settings.conf:/etc/php8/php-fpm.d/server.conf" andergrim/alpine-php-fpm-nginx

_Note; Because `-v` requires an absolute path I've added `pwd` in the example to return the absolute path to the current directory_

## Environment variables

You can define the next environment variables to change values from NGINX and PHP

| Server | Variable Name           | Default | description                                                                                                                                                                                                                                            |
|--------|-------------------------|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| NGINX  | client_max_body_size    | 2m      | Sets the maximum allowed size of the client request body, specified in the “Content-Length” request header field.                                                                                                                                      |
| PHP8   | clear_env               | no      | Clear environment in FPM workers. Prevents arbitrary environment variables from reaching FPM worker processes by clearing the environment in workers before env vars specified in this pool configuration are added.                                   |
| PHP8   | allow_url_fopen         | On      | Enable the URL-aware fopen wrappers that enable accessing URL object like files. Default wrappers are provided for the access of remote files using the ftp or http protocol, some extensions like zlib may register additional wrappers.              |
| PHP8   | allow_url_include       | Off     | Allow the use of URL-aware fopen wrappers with the following functions: include(), include_once(), require(), require_once().                                                                                                                          |
| PHP8   | display_errors          | Off     | Eetermine whether errors should be printed to the screen as part of the output or if they should be hidden from the user.                                                                                                                              |
| PHP8   | file_uploads            | On      | Whether or not to allow HTTP file uploads.                                                                                                                                                                                                             |
| PHP8   | max_execution_time      | 0       | Maximum time in seconds a script is allowed to run before it is terminated by the parser. This helps prevent poorly written scripts from tying up the server. The default setting is 30.                                                               |
| PHP8   | max_input_time          | -1      | Maximum time in seconds a script is allowed to parse input data, like POST, GET and file uploads.                                                                                                                                                      |
| PHP8   | max_input_vars          | 1000    | Maximum number of input variables allowed per request and can be used to deter denial of service attacks involving hash collisions on the input variable names.                                                                                        |
| PHP8   | memory_limit            | 128M    | Maximum amount of memory in bytes that a script is allowed to allocate. This helps prevent poorly written scripts for eating up all available memory on a server. Note that to have no memory limit, set this directive to -1.                         |
| PHP8   | post_max_size           | 8M      | Max size of post data allowed. This setting also affects file upload. To upload large files, this value must be larger than upload_max_filesize. Generally speaking, memory_limit should be larger than post_max_size.                                 |
| PHP8   | upload_max_filesize     | 2M      | Maximum size of an uploaded file.                                                                                                                                                                                                                      |
| PHP8   | zlib_output_compression | On      | Whether to transparently compress pages. If this option is set to "On" in php.ini or the Apache configuration, pages are compressed if the browser sends an "Accept-Encoding: gzip" or "deflate" header.                                               |

_Note; Because `-v` requires an absolute path I've added `pwd` in the example to return the absolute path to the current directory_


## Adding composer

If you need [Composer](https://getcomposer.org/) in your project, here's an easy way to add it.

```dockerfile
FROM andergrim/alpine-php-fpm-nginx:latest
USER root
# Install composer from the official image
RUN apk add --no-cache composer
USER nobody
# Run composer install to install the dependencies
RUN composer install --optimize-autoloader --no-interaction --no-progress
```

### Building with composer

If you are building an image with source code in it and dependencies managed by composer then the definition can be improved.
The dependencies should be retrieved by the composer but the composer itself (`/usr/bin/composer`) is not necessary to be included in the image.

```Dockerfile
FROM composer AS composer

# copying the source directory and install the dependencies with composer
COPY <your_directory>/ /app

# run composer install to install the dependencies
RUN composer install \
  --optimize-autoloader \
  --no-interaction \
  --no-progress

# continue stage build with the desired image and copy the source including the
# dependencies downloaded by composer
FROM andergrim/alpine-php-fpm-nginx
COPY --chown=nginx --from=composer /app /var/www/html
```
