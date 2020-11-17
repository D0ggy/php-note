[toc]

# Building PHP extensions

Now that you know how to compile PHP itself, we’ll move on to compiling additional extensions. We’ll discuss how the build process works and what different options are available.

现在您已经知道如何编译 PHP，我们将继续编译其他扩展。我们将讨论构建过程的工作原理以及可用的其他选项。

## <span id="loading-shared-extensions">Loading shared extensions</span>

As you already know from the previous section, PHP extensions can be either built statically into the PHP binary, or compiled into a shared object (`.so`). Static linkage is the default for most of the bundled extensions, whereas shared objects can be created by explicitly passing `--enable-EXTNAME=shared` or `--with-EXTNAME=shared` to `./configure`.

如您在上一节中已经知道的那样，PHP 扩展可以静态地内置到 PHP 二进制文件中，也可以编译成共享对象（`.so`）。静态链接是大多数捆绑扩展的默认设置，而共享对象可以通过将 `--enable-EXTNAME=shared` 或`--with-EXTNAME=shared` 显式传递给 `./configure` 来创建。

While static extensions will always be available, shared extensions need to be loaded using the `extension` or `zend_extension` ini options. Both options take either an absolute path to the `.so` file or a path relative to the `extension_dir` setting.

虽然静态扩展总是可用的，但是共享扩展需要使用 `extension` 或 `zend_extension` ini选项加载。这两个选项都采用 `.so` 文件的绝对路径或相对于 `extension_dir` 设置的路径。

As an example, consider a PHP build compiled using this configure line:

```bash
~/php-src> ./configure --prefix=$HOME/myphp \
                       --enable-debug --enable-maintainer-zts \
                       --enable-opcache --with-gmp=shared
```

In this case both the opcache extension and GMP extension are compiled into shared objects located in the `modules/` directory. You can load both either by changing the `extension_dir` or by passing absolute paths:

在这种情况下，opcache 扩展和 GMP 扩展都被编译为位于 `modules/` 目录中的共享对象。您可以通过更改 `extension_dir` 或通过传递绝对路径来加载：

```bash
~/php-src> sapi/cli/php -dzend_extension=`pwd`/modules/opcache.so \
                        -dextension=`pwd`/modules/gmp.so
# or
~/php-src> sapi/cli/php -dextension_dir=`pwd`/modules \
                        -dzend_extension=opcache.so -dextension=gmp.so
```

During the `make install` step, both `.so` files will be moved into the extension directory of your PHP installation, which you may find using the `php-config --extension-dir` command. For the above build options it will be `/home/myuser/myphp/lib/php/extensions/no-debug-non-zts-MODULE_API`. This value will also be the default of the `extension_dir` ini option, so you won’t have to specify it explicitly and can load the extensions directly:

在 `make install` 步骤中，两个 `.so` 文件都将移到 PHP 安装的扩展目录中，您可以使用 `php-config --extension-dir` 命令找到该目录。对于上述构建选项，它的目录将为 `/home/myuser/myphp/lib/php/extensions/no-debug-non-zts-MODULE_API`。此值也将是 `extension_dir` ini选项的默认值，因此您不必明确指定它，并且可以直接加载扩展名：

```bash
~/myphp> bin/php -dzend_extension=opcache.so -dextension=gmp.so
```

This leaves us with one question: Which mechanism should you use? Shared objects allow you to have a base PHP binary and load additional extensions through the php.ini. Distributions make use of this by providing a bare PHP package and distributing the extensions as separate packages. On the other hand, if you are compiling your own PHP binary, you likely don’t have need for this, because you already know which extensions you need.

这给我们留下了一个问题：您应该使用哪种机制？共享对象允许您可以使用基础的 PHP 二进制文件和通过 php.ini 加载其他扩展。发行版通过提供一个裸露的 PHP 包并将这些扩展作为单独的包分发来利用此功能。另一方面，如果您要编译自己的PHP二进制文件，则可能不需要它，因为您已经知道需要哪些扩展。

As a rule of thumb, you’ll use static linkage for the extensions bundled by PHP itself and use shared extensions for everything else. The reason is simply that building external extensions as shared objects is easier (or at least less intrusive), as you will see in a moment. Another benefit is that you can update the extension without rebuilding PHP.

根据经验，您将对 PHP 本身捆绑的扩展使用静态链接，对其他所有扩展使用共享扩展。原因很简单，就像稍后将看到的那样，将外部扩展构建为共享对象更加容易（或至少减少了侵入性）。另一个好处是您可以在不重建PHP的情况下更新扩展名。

>**Note**
If you need information about the difference between extensions and Zend extensions, you ==may have a look at the dedicated chapter==.

## <span id="installing-extensions-from-pecl">Installing extensions from PECL</share>

[PECL](http://pecl.php.net/), the _PHP Extension Community Library_, offers a large number of extensions for PHP. When extensions are removed from the main PHP distribution, they usually continue to exist in PECL. Similarly many extensions that are now bundled with PHP were previously PECL extensions.

[PECL](http://pecl.php.net/),_PHP 扩展社区库_ 提供了许多 PHP 扩展。从主 PHP 发行版中删除的扩展，它们通常继续存在于 PECL 中。同样，现在与 PHP 捆绑在一起的许多扩展以前都是 PECL 扩展。

Unless you specified `--without-pear` during the configuration stage of your PHP build, `make install` will download and install PECL as a part of PEAR. You will find the `pecl` script in the `$PREFIX/bin` directory. Installing extensions is now as simple as running `pecl install EXTNAME`, e.g.:

除非在PHP构建的配置阶段指定了 `--with-pear`，否则 `make install` 将下载并安装 PECL 作为 PEAR 的一部分。您可以在 `$PREFIX/bin` 目录中找到 `pecl` 脚本。现在，安装扩展程序就像运行 `pecl install EXTNAME` 一样简单，例如：

```bash
~/myphp> bin/pecl install apcu
```

This command will download, compile and install the [APCu](http://pecl.php.net/package/APCu) extension. The result will be a `apcu.so` file in your extension directory, which can then be loaded by passing the `extension=apcu.so` ini option.

此命令将下载，编译和安装 [APCu](http://pecl.php.net/package/APCu) 扩展。结果是扩展目录中生成 `apcu.so` 文件，然后可以通过传递给 ini 中的 `extension=apcu.so` 选项来加载该文件。

While `pecl install` is very handy for the end-user, it is of little interest to extension developers. In the following, we’ll describe two ways to manually build extensions: Either by importing it into the main PHP source tree (this allows static linkage) or by doing an external build (only shared).

尽管 `pecl install` 对于终端用户非常方便，但是扩展开发人员对此并不感兴趣。在下文中，我们将介绍两种手动构建扩展的方法：通过将其导入主 PHP 源代码树（这允许静态链接）或进行外部构建（仅共享）。

## <span id="adding-extensions-to-the-php-source-tree">Adding extensions to the PHP source tree<span>

There is no fundamental difference between a third-party extension and an extension bundled with PHP. As such you can build an external extension simply by copying it into the PHP source tree and then using the usual build procedure. We’ll demonstrate this using APCu as an example.

第三方扩展和与 PHP 捆绑在一起的扩展之间没有根本区别。这样，您可以简单地通过将复制到PHP源代码树中构建外部扩展，然后使用通常的构建过程来构建外部扩展。我们将以 APCu 为例进行演示。

First of all, you’ll have to place the source code of the extension into the `ext/EXTNAME` directory of your PHP source tree. If the extension is available via git, this is as simple as cloning the repository from within `ext/`:

首先，您必须将扩展程序的源代码放入 PHP 源代码树的 `ext/EXTNAME` 目录中。如果扩展可以通过 git 获得，那么就像从 `ext/` 中克隆存储库一样简单：

```bash
~/php-src/ext> git clone https://github.com/krakjoe/apcu.git
```
Alternatively you can also download a source tarball and extract it:

另外，您也可以下载源 tarball 并将其解压缩：

```bash
/tmp> wget http://pecl.php.net/get/apcu-4.0.2.tgz
/tmp> tar xzf apcu-4.0.2.tgz
/tmp> mkdir ~/php-src/ext/apcu
/tmp> cp -r apcu-4.0.2/. ~/php-src/ext/apcu
```
The extension will contain a `config.m4` file, which specifies extension-specific build instructions for use by autoconf. To incorporate them into the `./configure` script, you’ll have to run `./buildconf` again. To ensure that the configure file is really regenerated, it is recommended to delete it beforehand:

该扩展名将包含 `config.m4` 文件，该文件指定供 autoconf 使用的特定于扩展名的构建指令。 要将它们合并到./configure脚本中，您必须再次运行 `./buildconf`。 为了确保确实重新生成了配置文件，建议事先将其删除：

```bash
~/php-src> rm configure && ./buildconf --force
```
You can now use the `./config.nice` script to add APCu to your existing configuration or start over with a completely new configure line:

现在，您可以使用 `./config.nice` 脚本将 APCu 添加到现有配置中，或者从全新的配置行开始：

```bash
~/php-src> ./config.nice --enable-apcu
# or
~/php-src> ./configure --enable-apcu # --other-options
```
Finally run `make -jN` to perform the actual build. As we didn’t use `--enable-apcu=shared` the extension is statically linked into the PHP binary, i.e. no additional actions are needed to make use of it. Obviously you can also use `make install` to install the resulting binaries.

最后运行 `make -jN` 来执行实际的构建。由于我们没有使用 `--enable-apcu=shared`，因此扩展名已静态链接到PHP二进制文件中，即无需其他操作即可使用它。显然，您也可以使用 `make install` 来安装生成的二进制文件。

## <span id="building-extensions-using-phpize">Building extensions using phpize<span>

It is also possible to build extensions separately from PHP by making use of the `phpize` script that was already mentioned in the [Building PHP](../_1_Building_PHP/README.md#building-php) section.

通过使用[Building PHP](../_1_Building_PHP/README.md#building-php) 中已经提到的 `phpize` 脚本，可以使构建扩展与 PHP 分开。

`phpize` plays a similar role as the `./buildconf` script used for PHP builds: First it will import the PHP build system into your extension by copying files from `$PREFIX/lib/php/build`. Among these files are `acinclude.m4` (PHP’s M4 macros), `phpize.m4` (which will be renamed to `configure.in` in your extension and contains the main build instructions) and `run-tests.php`.

`phpize` 与用于 PHP 构建的 `./buildconf` 脚本扮演者相同的角色：首先，它将通过从 `$PREFIX/lib/php/build` 复制文件将PHP构建系统导入您的扩展程序。这些文件中包括 `acinclude.m4` （PHP 的 M4 宏），`phpize.m4` （将在您的扩展程序中重命名为 `configure.in` 并包含主要的构建说明）和 `run-tests.php`。

Then `phpize` will invoke autoconf to generate a `./configur`e file, which can be used to customize the extension build. Note that it is not necessary to pass `--enable-apcu` to it, as this is implicitly assumed. Instead you should use `--with-php-config` to specify the path to your `php-config` script:

然后 `phpize` 将调用 `autoconf` 生成一个 `./configure` 文件，该文件可用于自定义扩展构建。注意，没有必要将 `--enable-apcu` 传递给它，因为这是隐式假定的。相反，您应该使用 `--with-php-config` 来指定 `php-config` 脚本的路径：

```bash
/tmp/apcu-4.0.2> ~/myphp/bin/phpize
Configuring for:
PHP Api Version:         20121113
Zend Module Api No:      20121113
Zend Extension Api No:   220121113

/tmp/apcu-4.0.2> ./configure --with-php-config=$HOME/myphp/bin/php-config
/tmp/apcu-4.0.2> make -jN && make install
```
You should always specify the `--with-php-config` option when building extensions (unless you have only a single, global installation of PHP), otherwise `./configure` will not be able to correctly determine what PHP version and flags to build against. Specifying the `php-config` script also ensures that `make install` will move the generated `.so` file (which can be found in the `modules/` directory) to the right extension directory.

构建扩展时，应始终指定 `--with-php-config` 选项（除非只有一个全局的PHP安装），否则 `./configure` 将无法正确确定要针对哪个 PHP 版本和标志进行构建。指定 `php-config` 脚本还可以确保 `make install` 将生成的 `.so` 文件（可以在 `modules/` 目录中找到）移动到正确的扩展目录中。

As the `run-tests.php` file was also copied during the `phpize` stage, you can run the extension tests using `make test` (or an explicit call to run-tests).

由于也在 `phpize` 阶段复制了 `run-tests.php` 文件，因此您可以使用 `make test`（或对 run-tests 的显式调用）运行扩展测试。

The `make clean` target for removing compiled objects is also available and allows you to force a full rebuild of the extension, should the incremental build fail after a change. Additionally phpize provides a cleaning option via `phpize --clean`. This will remove all the files imported by `phpize`, as well as the files generated by the `/configure` script.

用于清除已编译对象的 `make clean` 是可用的并且允许您强制重新构建扩展，在更改后增量构建失败。另外 `phpize` 通过 `phpize --clean` 提供了一个清洁选项。这将删除 `phpize` 导入的所有文件以及 `/configure` 脚本生成的文件。

## <span id="displaying-information-about-extensions">Displaying information about extensions</span>

The PHP CLI binary provides several options to display information about extensions. You already know `-m`, which will list all loaded extensions. You can use it to verify that an extension was loaded correctly:

PHP CLI 二进制文件提供了几个选项来显示有关扩展的信息。 您已经知道 `-m`，它将列出所有已加载的扩展名。 您可以使用它来验证扩展名是否正确加载：
```bash
~/myphp/bin> ./php -dextension=apcu.so -m | grep apcu
apcu
```
There are several further switches beginning with `--r` that expose Reflection functionality. For example you can use `--ri` to display the configuration of an 
extension:

还有一些以 `--r` 开头的功能开关展露 Reflection 类的功能。例如可以用  `--ri` 展示扩展的配置：

```bash
~/myphp/bin> ./php -dextension=apcu.so --ri apcu
apcu

APCu Support => disabled
Version => 4.0.2
APCu Debugging => Disabled
MMAP Support => Enabled
MMAP File Mask =>
Serialization Support => broken
Revision => $Revision: 328290 $
Build Date => Jan  1 2014 16:40:00

Directive => Local Value => Master Value
apc.enabled => On => On
apc.shm_segments => 1 => 1
apc.shm_size => 32M => 32M
apc.entries_hint => 4096 => 4096
apc.gc_ttl => 3600 => 3600
apc.ttl => 0 => 0
# ...
```
The `--re` switch lists all ini settings, constants, functions and classes added by an extension:

`--re` 开关列出了扩展添加的所有 ini 设置，常量，函数和类：

```bash
~/myphp/bin> ./php -dextension=apcu.so --re apcu
Extension [ <persistent> extension #27 apcu version 4.0.2 ] {
  - INI {
    Entry [ apc.enabled <SYSTEM> ]
      Current = '1'
    }
    Entry [ apc.shm_segments <SYSTEM> ]
      Current = '1'
    }
    # ...
  }

  - Constants [1] {
    Constant [ boolean APCU_APC_FULL_BC ] { 1 }
  }

  - Functions {
    Function [ <internal:apcu> function apcu_cache_info ] {

      - Parameters [2] {
        Parameter #0 [ <optional> $type ]
        Parameter #1 [ <optional> $limited ]
      }
    }
    # ...
  }
}
```

The `--re` switch only works for normal extensions, Zend extensions use `--rz` instead. You can try this on opcache:

`--re` 开关仅适用于普通扩展名，Zend 扩展名可以使用 `--rz` 代替。 您可以在opcache上尝试：

```bash
~/myphp/bin> ./php -dzend_extension=opcache.so --rz "Zend OPcache"
Zend Extension [ Zend OPcache 7.0.3-dev Copyright (c) 1999-2013 by Zend Technologies <http://www.zend.com/> ]
```

As you can see, this doesn’t display any useful information. The reason is that opcache registers both a normal extension and a Zend extension, where the former contains all ini settings, constants and functions. So in this particular case you still need to use `--re`. Other Zend extensions make their information available via `--rz` though.

如您所见，它不会显示任何有用的信息。原因是opcache同时注册了普通扩展名和 Zend 扩展名，其中前者包含所有 ini 设置，常量和函数。因此，在这种情况下，您仍然需要使用 `--re`。但是，其他 Zend 扩展通过 `--rz` 使其信息可用。

## <span id="extensions-api-compatibility">Extensions API compatibility<span>

Extensions are very sensitive to 5 major factors. If they don’t fit, the extension won’t load into PHP and will be useless:

扩展对 5 个主要因素非常敏感。如果不合适，则该扩展将不会加载到 PHP 中，并且将无用：

- PHP Api Version
- Zend Module Api No
- Zend Extension Api No
- Debug mode
- Thread safety

The _phpize_ tool recall you some of those information. So if you have built a PHP with debug mode, and try to make it load and use an extension which has been built without debug mode, it simply won’t work. Same for the other checks.

_phpize_ 工具可以使您回忆一些信息。因此，如果您构建了具有调试模式的PHP，并尝试使其加载并使用没有调试模式构建的扩展程序，那么它将根本无法工作。其他检查也一样。

_PHP Api Version_ is the number of the version of the internal API._Zend Module Api No_ and Zend Extension Api No are respectively about PHP extensions and Zend extensions API.

_PHP Api Version_ 是内部API版本的编号。_Zend Module Api No_ 和 _Zend Extension Api No_ 分别与 PHP 扩展和 Zend 扩展 API 有关。

Those numbers are later passed as C macros to the extension being built, so that it can itself check against those parameters and take different code paths based on C preprocessor `#ifdefs`. As those numbers are passed to the extension code as macros, they are written in the extension structure, so that anytime you try to load this extension in a PHP binary, they will be checked against the PHP binary’s own numbers. If they mismatch, then the extension will not load, and an error message will be displayed.

这些数字随后作为 C 宏传递给正在构建的扩展，以便它本身可以检查这些参数并基于 C 预处理程序 `#ifdefs` 采用不同的代码路径。当这些数字作为宏传递到扩展代码时，它们会以扩展结构的形式编写，因此，每当您尝试将此扩展加载到 PHP 二进制文件中时，都将对照 PHP 二进制文件本身的数字进行检查。如果它们不匹配，则分机将不会加载，并且将显示一条错误消息。

If we look at the extension C structure, it looks like this:
```bash
zend_module_entry foo_module_entry = {
    STANDARD_MODULE_HEADER,
    "foo",
    foo_functions,
    PHP_MINIT(foo),
    PHP_MSHUTDOWN(foo),
    NULL,
    NULL,
    PHP_MINFO(foo),
    PHP_FOO_VERSION,
    STANDARD_MODULE_PROPERTIES
};
```
What is interesting for us so far, is the `STANDARD_MODULE_HEADER` macro. If we expand it, we can see:

到目前为止，对我们来说有趣的是 `STANDARD_MODULE_HEADER` 宏。 如果我们扩展它，我们可以看到：
```c
#define STANDARD_MODULE_HEADER_EX sizeof(zend_module_entry), ZEND_MODULE_API_NO, ZEND_DEBUG, USING_ZTS
#define STANDARD_MODULE_HEADER STANDARD_MODULE_HEADER_EX, NULL, NULL
```

Notice how `ZEND_MODULE_API_NO`, `ZEND_DEBUG`, `USING_ZTS` are used.

If you look at the default directory for PHP extensions, it should look like `no-debug-non-zts-20090626`. As you’d have guessed, this directory is made of distinct parts joined together : debug mode, followed by thread safety information, followed by the Zend Module Api No. So by default, PHP tries to help you navigating with extensions.

如果查看 PHP 扩展的默认目录，则其外观应类似于 `no-debug-non-zts-20090626`。 您可能已经猜到，该目录由不同的部分组成：调试模式，后跟线程安全信息，后跟 Zend Module ApiNo。因此，默认情况下，PHP 会尝试帮助您使用扩展进行导航。

>**Note**
Usually, when you become an internal developer or an extension developer, you will have to play with the debug parameter, and if you have to deal with the Windows platform, threads will show up as well. You can end with compiling the same extension several times against several cases of those parameters.

>**注意**
通常，当您成为内部开发人员或扩展开发人员时，必须使用 debug 参数，并且如果必须使用 Windows 平台，线程也会显示出来。 您可以针对这些参数的几种情况多次编译同一扩展名。

Remember that every new major/minor version of PHP change parameters such as the PHP Api Version, that’s why you need to recompile extensions against a newer PHP version.

请记住，每个新的主要/次要版本的 PHP 都会更改参数（例如 PHP Api 版本），因此您需要针对较新的 PHP 版本重新编译扩展。 

```bash
> /path/to/php70/bin/phpize -v
Configuring for:
PHP Api Version:         20151012
Zend Module Api No:      20151012
Zend Extension Api No:   320151012

> /path/to/php71/bin/phpize -v
Configuring for:
PHP Api Version:         20160303
Zend Module Api No:      20160303
Zend Extension Api No:   320160303

> /path/to/php56/bin/phpize -v
Configuring for:
PHP Api Version:         20131106
Zend Module Api No:      20131226
Zend Extension Api No:   220131226
```

>**Note**
_Zend Module Api No_ is itself built with a date using the year.month.day format. This is the date of the day the API changed and was tagged. _Zend Extension Api No_ is the Zend version followed by _Zend Module Api No_.

>**注意**
_Zend Module Api No_ 本身使用使用 year.month.day 格式的日期构建。这是 API 更改并被标记的日期。 _Zend Extension Api No_ 是Zend版本，其后是 _Zend Module Api No_。

>**Note**
Too many numbers? Yes. One API number, bound to one PHP version, would really be enough for anybody and would ease the understanding of PHP versioning. Unfortunately, we got 3 different API numbers in addition to the PHP version itself. Which one should you look for? The answer is any : they all-three-of-them evolve when PHP version evolve. For historical reasons, we got 3 different numbers.

>**注意**
数字太多？ 是。 一个 API 编号，绑定到一个 PHP 版本，对于任何人来说确实足够，并且可以简化对 PHP 版本控制的理解。不幸的是，除了PHP版本本身，我们还获得了3个不同的API编号。您应该寻找哪一个？ 答案是任意的：当 PHP 版本演变时，它们全部由三者演变而成。 由于历史原因，我们得到了3个不同的数字。

But, you are a C developer anren’t you? Why not build a “compatibility” header on your side, based on such number? We authors, use something like this in extensions of ours:

但是，您是  C开发人员，不是吗？为什么不根据这样的数字在您这一边建立“兼容性”标头？ 我们在我们的扩展中使用以下内容：
```c
#include "php.h"
#include "Zend/zend_extensions.h"

#define PHP_5_5_X_API_NO            220121212
#define PHP_5_6_X_API_NO            220131226

#define PHP_7_0_X_API_NO            320151012
#define PHP_7_1_X_API_NO            320160303
#define PHP_7_2_X_API_NO            320160731

#define IS_PHP_72          ZEND_EXTENSION_API_NO == PHP_7_2_X_API_NO
#define IS_AT_LEAST_PHP_72 ZEND_EXTENSION_API_NO >= PHP_7_2_X_API_NO

#define IS_PHP_71          ZEND_EXTENSION_API_NO == PHP_7_1_X_API_NO
#define IS_AT_LEAST_PHP_71 ZEND_EXTENSION_API_NO >= PHP_7_1_X_API_NO

#define IS_PHP_70          ZEND_EXTENSION_API_NO == PHP_7_0_X_API_NO
#define IS_AT_LEAST_PHP_70 ZEND_EXTENSION_API_NO >= PHP_7_0_X_API_NO

#define IS_PHP_56          ZEND_EXTENSION_API_NO == PHP_5_6_X_API_NO
#define IS_AT_LEAST_PHP_56 (ZEND_EXTENSION_API_NO >= PHP_5_6_X_API_NO && ZEND_EXTENSION_API_NO < PHP_7_0_X_API_NO)

#define IS_PHP_55          ZEND_EXTENSION_API_NO == PHP_5_5_X_API_NO
#define IS_AT_LEAST_PHP_55 (ZEND_EXTENSION_API_NO >= PHP_5_5_X_API_NO && ZEND_EXTENSION_API_NO < PHP_7_0_X_API_NO)

#if ZEND_EXTENSION_API_NO >= PHP_7_0_X_API_NO
#define IS_PHP_7 1
#define IS_PHP_5 0
#else
#define IS_PHP_7 0
#define IS_PHP_5 1
#endif
```
See?

Or, simpler (so better) is to use `PHP_VERSION_ID` which you are probably much more familiar about:
```c
#if PHP_VERSION_ID >= 50600
```