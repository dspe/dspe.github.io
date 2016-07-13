# Opcache Tips

With new version of PHP, APC has been replace with OPCache to increase performances. OPCache is not only an byte-code caching but an optimization tools too. If you're curious about the benefit of OpCache you could read [this blog post](https://blog.appdynamics.com/php/why-every-php-application-should-use-an-opcache/).

There are 3 relevant parameters to play with:
- **opcache.memory_consumption**: memory used for caching (default is 64MB)
- **opcache.max_accelerated_files**: number of files could be cacheable (by default is 2000 and max is 100000)
- **opcache.max_wasted_percentage**: maximum percentage of wasted memory (by default is 5%)
- **opcache.validate_timestamps**: if disabled, you need to reset the OPCache manually or restart the webserver for changes to the filesystem to take effect.

As you could imagine, each parameters will depends of your project and of your environment: of course you will not disable the *validate_timestamp* on a developer server, that's really make no sence.

Above you could have an example of configuration already used:

```
zend_extension=opcache.so
opcahe.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=12
opcache.max_accelerated_files=10000
opcache.max_wasted_percentage=4
; avoid unnecessary filesystem calls
opcache.validate_timestamps=0
; if you are in dev environment you need to modify and enable this value
; determine in second how ofer should the code cache expire
; disable in production due to the parameter validate_timestamps
;opcache.revalidate_freq=10
opcache.save_comments=0
opcache.load_comments=0
; if you have segfault remove this value
opcache.fast_shutdown=1
opcache.enable_file_override=1
opcache.error_log=/var/log/php5/php5-opcache.error.log
; fatal errors (level 0), errors (level 1)
; warnings (level 2), info messages (level 3)
; debug messages (level 4)
opcache.log_verbosity_level=3
```

NB: to find a good value for ```opcache.max_accelerated_files``` you can use this bash command:

```
$ find / -iname *.php|wc -l
```

You could find some OpCache GUI to monitor the behavior. Some example:
- https://github.com/amnuts/opcache-gui
- https://github.com/rlerdorf/opcache-status
