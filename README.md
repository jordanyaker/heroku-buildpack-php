Respage Apache+PHP build pack
========================

This is a build pack bundling PHP and Apache for Heroku apps.

Configuration
-------------

The config files are bundled with the build pack itself:

* conf/httpd.conf
* conf/php.ini


Pre-compiling binaries
----------------------

In order to make the installation of the respage server specialized to support the needs of the 4walls team, a custom set of binary tools was compiled and deployed to the S3 bucket located at `https://s3-us-west-2.amazonaws.com/respage-buildpacks/`.  The following script can be used, or modified, to reproduce the completed binaries.

    # an Ubuntu VM with version 10.04 was used for the compilation of this document.
    sudo apt-get -y update && sudo apt-get -y install g++ gcc libssl-dev libcloog-ppl0 libpng-dev libjpeg-dev libxml2-dev libmysqlclient-dev libpq-dev libpcre3-dev php5-dev php-pear  curl libcurl3 libcurl3-dev php5-curl libsasl2-dev libmagickwand-dev

    #download the source files for the compilation
    curl -L http://archive.apache.org/dist/httpd/httpd-2.2.25.tar.gz -o /tmp/httpd-2.2.25.tar.gz
    curl -L http://us.php.net/get/php-5.3.10.tar.gz/from/us2.php.net/mirror -o /tmp/php-5.3.10.tar.gz
    curl -L https://launchpad.net/libmemcached/1.0/1.0.4/+download/libmemcached-1.0.4.tar.gz -o /tmp/libmemcached-1.0.4.tar.gz
    curl -L http://pecl.php.net/get/memcached-2.0.1.tgz -o /tmp/memcached-2.0.1.tgz
    curl -L https://launchpad.net/imagemagick/main/6.8.6-6/+download/ImageMagick-6.8.6-6.tar.gz -o /tmp/ImageMagick-6.8.6-6.tar.gz
    curl -L http://pecl.php.net/get/imagick-3.0.1.tgz -o /tmp/imagick-3.0.1.tgz

    #untar all the srcs
    tar -C /tmp -xzvf /tmp/httpd-2.2.25.tar.gz
    tar -C /tmp -xzvf /tmp/php-5.3.10.tar.gz
    tar -C /tmp -xzvf /tmp/libmemcached-1.0.4.tar.gz
    tar -C /tmp -xzvf /tmp/memcached-2.0.1.tgz
    tar -C /tmp -xzvf /tmp/ImageMagick-6.8.6-6.tar.gz
    tar -C /tmp -xzvf /tmp/imagick-3.0.1.tgz

    #make the directories (Ubuntu requires sudo permissions for this step)
    sudo mkdir /app
    sudo mkdir /app/{apache,php,local}
    sudo mkdir /app/php/ext
    sudo mkdir /app/local/lib

    #copy libs
    sudo cp /usr/lib/libmysqlclient* /app/local/lib/
    sudo cp /usr/lib/libsasl2* /app/local/lib/

    # apache
    cd /tmp/httpd-2.2.25
    ./configure --prefix=/app/apache --enable-rewrite --enable-so --enable-deflate --enable-expires --enable-headers
    make && sudo make install

    # php
    cd /tmp/php-5.3.10
    ./configure --prefix=/app/php --with-apxs2=/app/apache/bin/apxs --with-mysql --with-pdo-mysql --with-pgsql --with-pdo-pgsql --with-iconv --with-gd --with-curl=/usr/lib --with-config-file-path=/app/php --enable-soap=shared --with-openssl --enable-mbstring --with-mhash --enable-pcntl --enable-mysqlnd --with-pear --with-mysqli --with-jpeg-dir=/usr/lib
    make && sudo make install
    
    # libmemcached
    cd /tmp/libmemcached-1.0.4
    ./configure --prefix=/app/local
    make && sudo make install

    # pecl memcached
    cd /tmp/memcached-2.0.1
    # edit config.m4 line 21 so no => yes ############### IMPORTANT!!! ###############
    sed -i -e '21 s/no, no/yes, yes/' /tmp/memcached-2.0.1/config.m4
    /app/php/bin/phpize
    ./configure --with-libmemcached-dir=/app/local/ --prefix=/app/php --with-php-config=/app/php/bin/php-config
    make && sudo make install
    
    # ImageMagick
    cd /tmp/ImageMagick-6.8.6-6.tar.gz
    ./configure --prefix=/app/local
    make && sudo make install

    # pecl imagick
    cd /tmp/imagick-3.0.1
    /app/php/bin/phpize
    ./configure --with-imagick=/app/local --prefix=/app/php --with-php-config=/app/php/bin/php-config
    make && sudo make install

    # pecl packages
    sudo /app/php/bin/pear config-set php_dir /app/php
    sudo /app/php/bin/pecl install memcache
    sudo /app/php/bin/pecl install apc   
    sudo /app/php/bin/pecl install imagick 
    
    # make it a little leaner
    sudo rm -rf /app/apache/manual/

    # package
    cd /app
    sudo echo '2.2.25' > apache/VERSION
    sudo tar -zcvf apache-2.2.25.tar.gz apache
    sudo echo '5.3.10' > php/VERSION
    sudo tar -zcvf php-5.3.10.tar.gz php local
