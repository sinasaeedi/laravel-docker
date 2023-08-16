FROM php:8.2-apache

# Persistent dependencies
RUN set -eux; \
    apt-get update; \
    apt-get install -y gnupg2

# Add MSSQL repository	
RUN set -eux; \
    curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -; \
    curl https://packages.microsoft.com/config/debian/11/prod.list > /etc/apt/sources.list.d/mssql-release.list; \
    apt-get update

# Install MSSQL Prerequisites and LDAP
RUN set -eux; \
    ACCEPT_EULA=Y apt-get install -y --allow-downgrades \
    zip \
    unzip \
    supervisor \
    libldap2-dev \
    msodbcsql18 \
    odbcinst=2.3.7 \
    odbcinst1debian2=2.3.7 \
    unixodbc-dev=2.3.7 \
    unixodbc=2.3.7 \
    libgmp-dev \
	libzip-dev \
    libpng-dev \
    libwebp-dev \
    libjpeg-dev

# Install PHP extensions
RUN set -eux; \
    docker-php-ext-install gd gmp ldap zip pdo_mysql mysqli

RUN set -eux; \
    pecl install sqlsrv pdo_sqlsrv

RUN set -eux; \
    docker-php-ext-enable sqlsrv pdo_sqlsrv opcache

# Cleaning
RUN rm -rf /tmp/pear; \
    apt-get clean; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	rm -rf /var/lib/apt/lists/*

# Enable opcache
RUN set -eux; \
	{ \
		echo 'opcache.memory_consumption=128'; \
		echo 'opcache.interned_strings_buffer=8'; \
		echo 'opcache.max_accelerated_files=4000'; \
		echo 'opcache.revalidate_freq=2'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini

# Error log and reporting
RUN set -eux; \
	{ \
		echo 'error_reporting = E_ERROR | E_WARNING | E_PARSE | E_CORE_ERROR | E_CORE_WARNING | E_COMPILE_ERROR | E_COMPILE_WARNING | E_RECOVERABLE_ERROR'; \
		echo 'display_errors = Off'; \
		echo 'display_startup_errors = Off'; \
		echo 'log_errors = On'; \
		echo 'error_log = /dev/stderr'; \
		echo 'log_errors_max_len = 1024'; \
		echo 'ignore_repeated_errors = On'; \
		echo 'ignore_repeated_source = Off'; \
		echo 'html_errors = Off'; \
	} > /usr/local/etc/php/conf.d/error-logging.ini

# File upload
RUN set -eux; \
	{ \
		echo 'file_uploads = On'; \
        echo 'memory_limit = 500M'; \
        echo 'upload_max_filesize = 500M'; \
        echo 'post_max_size = 500M'; \
        echo 'max_execution_time = 1000'; \
	} > /usr/local/etc/php/conf.d/file-upload.ini

# Setup supervisor
RUN set -eux; \
	{ \
        echo '[unix_http_server]'; \
        echo 'file=/var/run/supervisor.sock'; \
        echo 'chmod=0700'; \
        echo '[supervisord]'; \
        echo 'logfile=/var/log/supervisor/supervisord.log'; \
        echo 'pidfile=/var/run/supervisord.pid'; \
        echo 'childlogdir=/var/log/supervisor'; \
        echo 'nodaemon=true'; \
        echo '[rpcinterface:supervisor]'; \
        echo 'supervisor.rpcinterface_factory=supervisor.rpcinterface:make_main_rpcinterface'; \
        echo '[supervisorctl]'; \
        echo 'serverurl=unix:///var/run/supervisor.sock'; \
        echo '[include]'; \
        echo 'files = /etc/supervisor/conf.d/*.conf'; \
        echo '[program:apache2]'; \
        echo 'command=apache2-foreground'; \
	} >> /etc/supervisor/supervisord.conf

# Setup laravel worker
RUN set -eux; \
	{ \
        echo '[program:laravel-schedule-work]'; \
        echo 'process_name=%(program_name)s_%(process_num)02d'; \
        echo 'command=php /var/www/html/artisan queue:work'; \
        echo 'autostart=true'; \
        echo 'autorestart=true'; \
        echo 'numprocs=1'; \
        echo 'redirect_stderr=true'; \
        echo 'stdout_logfile=/var/log/supervisor/laravel-queue.log'; \
	} >> /etc/supervisor/conf.d/laravel-worker.conf

# Set public folder as start place
RUN sed -i 's/\/var\/www\/html/\/var\/www\/html\/public/g' /etc/apache2/sites-enabled/000-default.conf

# Enable Apache modules
RUN a2enmod rewrite expires

CMD ["/usr/bin/supervisord"]