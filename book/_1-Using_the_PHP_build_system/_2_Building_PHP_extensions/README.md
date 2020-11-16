[toc]

# Building PHP extensions

Now that you know how to compile PHP itself, we’ll move on to compiling additional extensions. We’ll discuss how the build process works and what different options are available.

## <span id="loading-shared-extensions">Loading shared extensions</span>

As you already know from the previous section, PHP extensions can be either built statically into the PHP binary, or compiled into a shared object (`.so`). Static linkage is the default for most of the bundled extensions, whereas shared objects can be created by explicitly passing `--enable-EXTNAME=shared` or `--with-EXTNAME=shared` to `./configure`.

While static extensions will always be available, shared extensions need to be loaded using the `extension` or `zend_extension` ini options. Both options take either an absolute path to the .so file or a path relative to the `extension_dir` setting.

As an example, consider a PHP build compiled using this configure line:

```sh
~/php-src> ./configure --prefix=$HOME/myphp \
                       --enable-debug --enable-maintainer-zts \
                       --enable-opcache --with-gmp=shared
```

In this case both the opcache extension and GMP extension are compiled into shared objects located in the `modules/` directory. You can load both either by changing the `extension_dir` or by passing absolute paths:

```sh
~/php-src> sapi/cli/php -dzend_extension=`pwd`/modules/opcache.so \
                        -dextension=`pwd`/modules/gmp.so
# or
~/php-src> sapi/cli/php -dextension_dir=`pwd`/modules \
                        -dzend_extension=opcache.so -dextension=gmp.so
```

During the `make install` step, both `.so` files will be moved into the extension directory of your PHP installation, which you may find using the `php-config --extension-dir` command. For the above build options it will be `/home/myuser/myphp/lib/php/extensions/no-debug-non-zts-MODULE_API`. This value will also be the default of the `extension_dir` ini option, so you won’t have to specify it explicitly and can load the extensions directly:

```sh
~/myphp> bin/php -dzend_extension=opcache.so -dextension=gmp.so
```

This leaves us with one question: Which mechanism should you use? Shared objects allow you to have a base PHP binary and load additional extensions through the php.ini. Distributions make use of this by providing a bare PHP package and distributing the extensions as separate packages. On the other hand, if you are compiling your own PHP binary, you likely don’t have need for this, because you already know which extensions you need.

As a rule of thumb, you’ll use static linkage for the extensions bundled by PHP itself and use shared extensions for everything else. The reason is simply that building external extensions as shared objects is easier (or at least less intrusive), as you will see in a moment. Another benefit is that you can update the extension without rebuilding PHP.

>**Note**
If you need information about the difference between extensions and Zend extensions, you ==may have a look at the dedicated chapter==.

## <span id="installing-extensions-from-pecl">Installing extensions from PECL</share>

[PECL](http://pecl.php.net/),