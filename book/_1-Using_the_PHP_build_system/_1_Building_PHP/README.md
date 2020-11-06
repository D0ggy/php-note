[toc]

# Building PHP
-----------------
This chapter explains how you can compile PHP in a way that is suitable for development of extensions or core modifications. We will only cover builds on Unixoid systems. If you wish to build PHP on Windows, you should take a look at the step-by-step build instructions in the PHP wiki [1].

本章解释您怎样能以适合开发扩展和内核修改的方式来编译 PHP。我们将只介绍 Unixoid 系统上的构建。如果您想要在 Windows 上构建 PHP, 您应该看一下 [step-by-step build instructions](https://wiki.php.net/internals/windows/stepbystepbuild_sdk_2)，[[1]](#id3)。

This chapter also provides an overview of how the PHP build system works and which tools it uses, but a detailed description is outside the scope of this book.

本章概述了 PHP 构建系统的工作方式和使用的工具，详细的描述不在本书讨论范围内。

<span id="id3">[1]</span> 免责声明: 对在 Windows 上编译 PHP 产生的后果概不负责。

## Why not use packages?
---------------------------
If you are currently using PHP, you likely installed it through your package manager, using a command like `sudo apt-get install php`. Before explaining the actual compilation you should first understand why doing your own compile is necessary and you can’t just use a prebuilt package. There are multiple reasons for this:

如果您当前正使用 PHP，以可能通过包管理器，类似 `sudo apt-get install php` 的命令安装的。在解释实际编译之前您应该首先明白为什么自己编译是重要的并且您不能只用预编译的包。有多个原因：

Firstly, the prebuilt package only contains the resulting binaries, but misses other things that are necessary to compile extensions, e.g. header files. This can be easily remedied by installing a development package, which is typically called `php-dev`. To facilitate debugging with valgrind or gdb one could additionally install debug symbols, which are usually available as another package called `php-dbg`.

首先，预编译包仅仅包含生成的二进制文件，错过了其他的重要的编译扩展的内容，例如头文件。可以通过安装开发包轻松解决，通常被称为 `php-dev`。为了便于使用 valgrind 或 gdb 进行调试，可以另外安装调试符号，这些符号通常作为另一个名为 `php-dbg` 的软件包提供。

But even if you install headers and debug symbols, you’ll still be working with a release build of PHP. This means that it will be built with high optimization level, which can make debugging very hard. Furthermore release builds will not generate warnings about memory leaks or inconsistent data structures. Additionally prebuilt packages don’t enable thread safety, which is very helpful during development.

但是即使您安装了头文件和调试符号，您仍将使用 PHP 的发行版。意味着将以最高优化等级构建 PHP，使得调试非常困难。此外，发行版本不会生成有关内存泄漏或数据结构不一致的警告。另外，预编译版本不能启用线程安全，这在开发中很有帮助。

Another issue is that nearly all distributions apply additional patches to PHP. In some cases these patches only contain minor changes related to configuration, but some distributions make use of highly intrusive patches like Suhosin. Some of these patches are known to introduce incompatibilities with low-level extensions like opcache.

另外一个问题是，几乎所有发行版本都为 PHP 应用了额外的补丁。在某些情况下，这些补丁仅包含了与配置相关的微小更改，但是某些发行版使用了 Suhosin 等高度侵入性的修补程序。已知一些补丁被介绍与低级扩展（例如 opcache）不兼容。

PHP only provides support for the software as provided on php.net and not for the distribution-modified versions. If you want to report bugs, submit patches or make use of our help channels for extension-writing, you should always work against the official PHP version. When we talk about “PHP” in this book, we’re always referring to the officially supported version.

PHP 仅对 [php.net](https://php.net/) 上提供的软件提供支持，而对发行版修改的版本不提供支持。如果您想报告错误，提交补丁或利用我们的帮助渠道进行扩展编写，则应始终对照正式的PHP版本进行工作。当我们在本书中谈论 “PHP” 时，我们总是指受官方支持的版本。

## Obtaining the source code
---------------------------
Before you can build PHP you first need to obtain its source code. There are two ways to do this: You can either download an archive from PHP’s download page or clone the git repository from git.php.net (or the mirror on Github).

在构建 PHP 前，您首先需要获取它的源码。这里有两种方法：您可以从 [PHP’s download page](https://php.net/downloads.php) 下载一个归档文件，或者从 [git.php.net](http://git.php.net/) 克隆 git 仓库（或者从 GitHub 的镜像）。

The build process is slightly different for both cases: The git repository doesn’t bundle a configure script, so you’ll need to generate it using the buildconf script, which makes use of autoconf. Furthermore the git repository does not contain a pregenerated parser, so you’ll also need to have bison installed.

两种构建过程有略微不同：git 仓库中没有捆绑一个 `configure` 脚本，所以您要用 `autoconf` 生成。此外，git 仓库中不包含预生成解析器，所以您也需要安装 bison。 

We recommend to checkout out the source code from git, because this will provide you with an easy way to keep your installation updated and to try your code with different versions. A git checkout is also required if you want to submit patches or pull requests for PHP.

我们推荐从 git 检出源码，因为这会使安装、更新和尝试不同的版本更简单。如果您想为 PHP 贡献代码或者拉取 PHP ，那么从 git 检出还是有必要的。

To clone the repository, run the following commands in your shell:

克隆仓库需要如下命令：

```shell
~> git clone http://git.php.net/repository/php-src.git
~> cd php-src
# by default you will be on the master branch, which is the current
# development version. You can check out a stable branch instead:
~/php-src> git checkout PHP-7.0
```

If you have issues with the git checkout, take a look at the `Git FAQ` on the PHP wiki. The Git FAQ also explains how to setup git if you want to contribute to PHP itself. Furthermore it contains instructions on setting up multiple working directories for different PHP versions. This can be very useful if you need to test your extensions or changes against multiple PHP versions and configurations.

如果您使用 git checkout 有问题，在 PHP wiki 浏览一下 [Git FAQ](https://wiki.php.net/vcs/gitfaq)。如果您想为 PHP 贡献，Git FAQ 还解释了怎设置 git。另外，它包含了关于设置不同版本的 PHP 的数明。如果您想针对多个 PHP 版本和配置测试扩展活更改，这也很有用。

Before continuing you should also install some basic build dependencies with your package manager (you’ll likely already have the first three installed by default):

在继续之前，您还应该在包管理器中安装一些基本的构建依赖项（默认情况下，您可能已经安装了前三个）：

* `gcc` or some other compiler suite.
* `libc-dev`, which provides the C standard library, including headers.
* `make`, which is the build-management tool PHP uses.
* `autoconf`, which is used to generate the configure script. 用来生成 `configure` 脚本。
    * 2.59 or higher (for PHP 7.0-7.1)
    * 2.64 or higher (for PHP 7.2)
    * 2.68 or higher (for PHP 7.3)
* `libtool`, which helps manage shared libraries.
* `bison` which is used to generate the PHP parser.用来生成 PHP 解释器。
    * 2.4 or higher (for PHP 7.0-7.3)
    * 3.0 or higher (for PHP 7.4)
* `re2c`, which is used to generate the PHP lexer. The re2c lexer generator was once an optional dependency when building PHP from the Git repository. In PHP > 7.3 branches the generated lexer files are not bundled in the Git repository anymore.
用来生成 PHP 词法分析器。从 Git 仓库构建 PHP 时，re2c lexer 生成器曾经是`可选的依赖项`。在 PHP > 7.3 分支中，生成的语法分析器文件不在捆绑在 Git 仓库中。

On Debian/Ubuntu you can install all these with the following command:

在 Debian/Ubuntu 您可以通过以下命令安装所有这些：

```shell
~/php-src> sudo apt-get install build-essential autoconf libtool bison re2c
```
Depending on the extensions that you enable during the `./configure` stage PHP will need a number of additional libraries. When installing these, check if there is a version of the package ending in `-dev` or `-devel` and install them instead. The packages without `dev` typically do not contain necessary header files. For example a default PHP build will require libxml, which you can install via the `libxml2-dev` package.

根据您启用的这些扩展，在 `./configure` 阶段 PHP 需要许多其他库。当安装这些时，请检查是否有以 `-dev` 或 `-devel` 结尾的软件包版本，然后安装它们。没有 `dev` 的软件通常不包含必要的头文件。例如，默认的 PHP 构建将需要 `libxml`，您可以安装 `libxml2-dev` 的包。

If you are using Debian or Ubuntu you can use `sudo apt-get build-dep php7` to install a large number of optional build-dependencies in one go. If you are only aiming for a default build, many of them will not be necessary though.

如果您使用的是 Debian 或 Ubuntu，您可以用 `sudo apt-get build-dep php7` 一次性安装大量的可选的构建依赖项。如果您仅针对默认构建，则其中许多将不是必须的。

## Build overview 构建概览
---------------------------

Before taking a closer look at what the individual build steps do, here are the commands you need to execute for a “default” PHP build:

再仔细研究各个步骤之前，这是您需要为 "defalt" PHP 的构建执行的命令：

```shell
~/php-src> ./buildconf     # only necessary if building from git
~/php-src> ./configure
~/php-src> make -jN
```

For a fast build, replace N with the number of CPU cores you have available (see `grep "cpu cores" /proc/cpuinfo`).

为了快速构建，用可用的 CPU 的核数替换掉 `N` （通过 `grep "cpu cores" /proc/cpuinfo` 查看）。

By default PHP will build binaries for the CLI and CGI SAPIs, which will be located at `sapi/cli/php` and `sapi/cgi/php-cgi` respectively. To check that everything went well, try running `sapi/cli/php -v`.

默认的 PHP 将构建为 CLI 和 CGI SAPIs 构建二进制文件，它们分别位于 `sapi/cli/php` 和 `sapi/cgi/php-cgi`。要检查一切正常，请尝试 `sapi/cli/php -v`。

Additionally you can run `sudo make install` to install PHP into `/usr/local`. The target directory can be changed by specifying a `--prefix` in the configuration stage:

另外，您可以运行 `sudo make install` 安装 PHP 到 `/usr/local`。这个目标目录可以在配置阶段（configuration stage）通过指定 `--prefix` 来修改： 

```shell
~/php-src> ./configure --prefix=$HOME/myphp
~/php-src> make -jN
~/php-src> make install
```
Here `$HOME/myphp` is the installation location that will be used during the `make install` step. Note that installing PHP is not necessary, but can be convenient if you want to use your PHP build outside of extension development.

这里 `$HOME/myphp` 是在 `make install` 步骤中将使用的安装位置。请注意，不必安装 PHP，但是如果要在扩展开发之外使用 PHP ，这会很方便。

Now lets take a closer look at the individual build steps!

现在，让我们仔细看看各个构建步骤！

## The *./buildconf* script （./buildconf 脚本）

If you are building from the git repository, the first thing you’ll have to do is run the `./buildconf` script. This script does little more than invoking the `build/build.mk` makefile, which in turn calls `build/build2.mk`.

如果您将从 git 仓库构建，第一件事是您必须运行 `./buildconf` 脚本。这个脚本不止调用 `build/build.mk` makefile，其他的还调用 `build/build2.mk`。

The main job of these makefiles is to run `autoconf` to generate the `./configure` script and `autoheader` to generate the `main/php_config.h.in` template. The latter file will be used by configure to generate the final configuration header file `main/php_config.h`.

这些 makefiles 主要工作是运行 `autoconf` 生成 `.configure` 脚本和 `autoheader` 去生成 `main/php_config.h.in` 模板。后面的文件用来生成最终的配置头文件 `main/php_config.h`。  

Both utilities produce their results from the `configure.in` file (which specifies most of the PHP build process), the `acinclude.m4` file (which specifies a large number of PHP-specific M4 macros) and the `config.m4` files of individual extensions and SAPIs (as well as a bunch of other m4 files).

这两个实用程序都从 `configure.in` 文件（指定了大多数 PHP 构建过程），`acinclude.m4` 文件（指定了许多 PHP 特定的 M4 宏）以及个别扩展名的和 SAPI （以及许多其他 m4 文件的 `config.m4` 文件。

The good news is that writing extensions or even doing core modifications will not require much interaction with the build system. You will have to write small `config.m4` files later on, but those usually just use two or three of the high-level macros that `acinclude.m4` provides. As such we will not go into further detail here.

好消息是写扩展或者核心修改不在需要和构建系统太多交互。稍后您要编写小的 `config.m4` 文件，但是这些文件通常仅使用 `acinclude.m4` 提供的两个或者三个高级宏。因此，我们将不在此进一步详细介绍。

The `./buildconf` script only has two options: `--debug` will disable warning suppression when calling autoconf and autoheader. Unless you want to work on the buildsystem, this option will be of little interest to you.

`./buildconf` 脚本只有两个选项：`--debug` 会在调用 `autoconf` 和 `autoheader` 时禁用警告抑制。除非您要在构建系统上工作，否则此选项对您没有用处。

The second option is `--force`, which will allow running `./buildconf` in release packages (e.g. if you downloaded the packaged source code and want to generate a new `./configure`) and additionally clear the configuration caches `config.cache` and `autom4te.cache/`.

第二个选项是 `--force`,它允许在发行包中运行 `./buildconf` (例如，如果您下载了打包的源代码并想要生成新的 `./configure`，并且还请除了配置缓存 `config.cache` 和 `autom4te.cache/`。

If you update your git repository using `git pull` (or some other command) and get weird errors during the `make` step, this usually means that something in the build configuration changed and you need to run `./buildconf --force`.

如果您用 `git pull` （或其他命令）更新您的 git 仓库，并在 `make` 步骤出现了奇怪的错误，这通常意味着构建配置中某些内容已经修改了，您需要运行 ./`buildconf --force`。

## The _./configure_ script

Once the `./configure` script is generated you can make use of it to customize your PHP build. You can list all supported options using `--help`:

生成 `./configure` 脚本后，您可以使用它来自定义PHP构建。 您可以使用 `--help` 列出所有受支持的选项：

```
~/php-src> ./configure --help | less
```
The first part of the help will list various generic options, which are supported by all autoconf-based configuration scripts. One of them is the already mentioned `--prefix=DIR`, which changes the installation directory used by `make install`. Another useful option is `-C`, which will cache the result of various tests in the `config.cache` file and speed up subsequent `./configure` calls. Using this option only makes sense once you already have a working build and want to quickly change between different configurations.

help 的第一部分将列出各种通用选项，所有基于 autoconf 的配置脚本都支持这些选项。其中之一是已提及的 `--prefix=DIR`，它可以更改 `make install` 使用的安装目录。另一个有用的选项是 `-C`，它将各种测试的结果缓存在 `config.cache` 文件中，并加速随后的 `./configure` 调用。使用这个选项只有在您已经有了可工作的构建并且想在不同配置之间快速改变时才合理。

Apart from generic autoconf options there are also many settings specific to PHP. For example, you can choose which extensions and SAPIs should be compiled using the `--enable-NAME` and `--disable-NAME` switches. If the extension or SAPI has external dependencies you need to use `--with-NAME` and `--without-NAME` instead. If a library needed by `NAME` is not located in the default location (e.g. because you compiled it yourself) you can specify its location using `--with-NAME=DIR`.

除了通用的 autoconf 选项之外，还有许多针对 PHP 的设置。例如，您可以使用 `--enable-NAME` 和 `--disable-NAME` 开关选择应编译的扩展名和 `SAPI`。如果扩展或 `SAPI`具有外部依赖关系，则需要改用 `--with-NAME` 和 `--without-NAME` 。如果被依赖的库 `NAME` 不在默认位置（例如，因为您自己编译了），则可以使用 `--with-NAME=DIR` 来指定其位置。

By default PHP will build the CLI and CGI SAPIs, as well as a number of extensions. You can find out which extensions your PHP binary contains using the `-m` option. For a default PHP 7.0 build the result will look as follows:

默认 PHP 会构建 CLI 和 CGI SAPIS，以及许多扩展。您可以使用 `-m` 选项找出您的 PHP 二进制文件包含哪些扩展。对于默认的 PHP7.0 构建的结果如下所示：

```
~/php-src> sapi/cli/php -m
[PHP Modules]
Core
ctype
date
dom
fileinfo
filter
hash
iconv
json
libxml
pcre
PDO
pdo_sqlite
Phar
posix
Reflection
session
SimpleXML
SPL
sqlite3
standard
tokenizer
xml
xmlreader
xmlwriter
```
If you now wanted to stop compiling the CGI SAPI, as well as the _tokenizer_ and _sqlite3_ extensions and instead enable _opcache_ and _gmp_, the corresponding configure command would be:

如果您想停止编译 CGI SAPI，以及 _tokenizer_ 和 _sqlite3_ 的扩展，而启用 _opcache_ 和 _gmp_，则对应的 configure 命令是：

```
~/php-src> ./configure --disable-cgi --disable-tokenizer --without-sqlite3 \
                       --enable-opcache --with-gmp
```

By default most extensions will be compiled statically, i.e. they will be part of the resulting binary. Only the opcache extension is shared by default, i.e. it will generate an `opcache.so` shared object in the `modules/` directory. You can compile other extensions into shared objects as well by writing `--enable-NAME=shared` or `--with-NAME=shared` (but not all extensions support this). We’ll talk about how to make use of shared extensions in the next section.

默认情况下，大多数扩展都是静态编译的，即它们将成为结果二进制文件的一部分。默认情况下，仅共享opcache扩展名，即它将在modules /目录中生成一个opcache.so共享对象。您还可以通过编写--enable-NAME = shared或--with-NAME = shared将其他扩展编译为共享对象（但并非所有扩展都支持此功能）。在下一节中，我们将讨论如何利用共享扩展。

To find out which switch you need to use and whether an extension is enabled by default, check `./configure --help`. If the switch is either `--enable-NAME` or `--with-NAME` it means that the extension is not compiled by default and needs to be explicitly enabled. `--disable-NAME` or `--without-NAME` on the other hand indicate an extension that is compiled by default, but can be explicitly disabled.

Some extensions are always compiled and can not be disabled. To create a build that only contains the minimal amount of extensions use the `--disable-all` option:
```
~/php-src> ./configure --disable-all && make -jN
~/php-src> sapi/cli/php -m
[PHP Modules]
Core
date
pcre
Reflection
SPL
standard
```
The `--disable-all` option is very useful if you want a fast build and don’t need much functionality (e.g. when implementing language changes). For the smallest possible build you can additionally specify the `--disable-cgi` switch, so only the CLI binary is generated.

There are two more switches, which you should **always** specify when developing extensions or working on PHP:

`--enable-debug` enables debug mode, which has multiple effects: Compilation will run with `-g` to generate debug symbols and additionally use the lowest optimization level `-O0`. This will make PHP a lot slower, but make debugging with tools like `gdb` more predictable. Furthermore debug mode defines the `ZEND_DEBUG` macro, which will enable various debugging helpers in the engine. Among other things memory leaks, as well as incorrect use of some data structures, will be reported.

`--enable-maintainer-zts` enables thread-safety. This switch will define the ZTS macro, which in turn will enable the whole TSRM (thread-safe resource manager) machinery used by PHP. Writing thread-safe extensions for PHP is very simple, but only if make sure to enable this switch. If you need more information about thread safety and global memory management in PHP, you should read the globals management chapter

On the other hand you should not use either of these options if you want to perform performance benchmarks for your code, as both can cause significant and asymmetrical slowdowns.

Note that `--enable-debug` and `--enable-maintainer-zts` change the ABI of the PHP binary, e.g. by adding additional arguments to many functions. As such, shared extensions compiled in debug mode will not be compatible with a PHP binary built in release mode. Similarly a thread-safe extension (ZTS) is not compatible with a non-thread-safe PHP build (NTS).

Due to the ABI incompatibility `make install` (and PECL install) will put shared extensions in different directories depending on these options:

- `$PREFIX/lib/php/extensions/no-debug-non-zts-API_NO` for release builds without ZTS
- `$PREFIX/lib/php/extensions/debug-non-zts-API_NO` for debug builds without ZTS
- `$PREFIX/lib/php/extensions/no-debug-zts-API_NO` for release builds with ZTS
- `$PREFIX/lib/php/extensions/debug-zts-API_NO` for debug builds with ZTS

The `API_NO` placeholder above refers to the `ZEND_MODULE_API_NO` and is just a date like `20100525`, which is used for internal API versioning.