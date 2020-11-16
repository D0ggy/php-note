[toc]

# Building PHP
-----------------
This chapter explains how you can compile PHP in a way that is suitable for development of extensions or core modifications. We will only cover builds on Unixoid systems. If you wish to build PHP on Windows, you should take a look at the step-by-step build instructions in the PHP wiki [1].

本章解释您怎样能以适合开发扩展和内核修改的方式来编译 PHP。我们将只介绍 Unixoid 系统上的构建。如果您想要在 Windows 上构建 PHP, 您应该看一下 [step-by-step build instructions](https://wiki.php.net/internals/windows/stepbystepbuild_sdk_2)，[[1]](#id3)。

This chapter also provides an overview of how the PHP build system works and which tools it uses, but a detailed description is outside the scope of this book.

本章概述了 PHP 构建系统的工作方式和使用的工具，详细的描述不在本书讨论范围内。

<span id="id3">[1]</span> 免责声明: 对在 Windows 上编译 PHP 产生的后果概不负责。

## <span id="why-not-use-packages">Why not use packages?</span>
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

## <span id="obtaining-the-source-code">Obtaining the source code</span>
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

## <span id="build-overview">Build overview</span> 构建概览
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

## <span id="build-script">The *./buildconf* script</span>（./buildconf 脚本）

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

如果您用 `git pull` （或其他命令）更新您的 git 仓库，并在 `make` 步骤出现了奇怪的错误，这通常意味着构建配置中某些内容已经修改了，您需要运行 `./buildconf --force`。

## <span id="configure-script">The _./configure_ script</span>

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

默认情况下，大多数扩展都是静态编译的，即它们将成为结果二进制文件的一部分。只有扩展 opcache 是默认共享的，它将在 `modules/` 目录中生成一个 `opcache.so` 共享对象。您可以编译其他扩展为共享对象通过 `--enable-NAME=shared` 或 `--with-NAME=shared` （不是所有扩展都支持此功能）。在下一节中，我们将讨论如何利用共享扩展。

To find out which switch you need to use and whether an extension is enabled by default, check `./configure --help`. If the switch is either `--enable-NAME` or `--with-NAME` it means that the extension is not compiled by default and needs to be explicitly enabled. `--disable-NAME` or `--without-NAME` on the other hand indicate an extension that is compiled by default, but can be explicitly disabled.

要想知道您需要使用哪个开关以及是否默认启用某个扩展，使用 `./configure --help` 检查。如果状态是 `--enable-NAME` 或者 `--with-NAME` 则意味着这个扩展默认不被编译并且需要显式的启用。另一方面，`--disable-NAME` 或 `--without-NAME` 表示默认情况下一个扩展已编译，但是可以显式的禁用扩展。

Some extensions are always compiled and can not be disabled. To create a build that only contains the minimal amount of extensions use the `--disable-all` option:

有些扩展总是编译的，不能禁用。为了构建只包含最少量扩展的版本，请使用 `--disable-all` 选项：

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

如果您想快速构建并且不需要太多功能（比如，在实现语言更改时），`--disable-all` 会非常有用。对于最小可能的构建，您可以额外的指定 `--disable-cgi` 开关，结果只会生成 CLI 二进制文件。

There are two more switches, which you should **always** specify when developing extensions or working on PHP:

有两个开关，在开发扩展或者用 PHP 工作时需要经常指定：

`--enable-debug` enables debug mode, which has multiple effects: Compilation will run with `-g` to generate debug symbols and additionally use the lowest optimization level `-O0`. This will make PHP a lot slower, but make debugging with tools like `gdb` more predictable. Furthermore debug mode defines the `ZEND_DEBUG` macro, which will enable various debugging helpers in the engine. Among other things memory leaks, as well as incorrect use of some data structures, will be reported.

`--enable-debug` 启用调试模式，它具有多种作用：`-g` 可以在编译运行时生成调试符号，并额外的使用最低的优化级别 `-O0`。这会使 PHP 变慢很多，但是使用 `gdb` 之类的工具进行调试时更加可预测。 此外，调试模式定义了 `ZEND_DEBUG` 宏，它将在引擎中启用各种调试助手。 除其他事项外，还将报告内存泄漏以及某些数据结构的不正确使用。

`--enable-maintainer-zts` enables thread-safety. This switch will define the `ZTS` macro, which in turn will enable the whole TSRM (thread-safe resource manager) machinery used by PHP. Writing thread-safe extensions for PHP is very simple, but only if make sure to enable this switch. If you need more information about thread safety and global memory management in PHP, you should read the globals management chapter ==TODOOOOOOOOOOOOOOOOO==

`--enable-maintainer-zts` 启用线程安全。 此开关将定义 `ZTS` 宏，该宏又将在 PHP 启用整个 TSRM （线程安全资源管理器）机制。 为 PHP 编写线程安全扩展非常简单，但前提是必须确保启用此开关。 如果您需要有关PHP中线程安全和全局内存管理的更多信息，则应阅读globals管理一章。==TODOOOOOOOOOO==

On the other hand you should not use either of these options if you want to perform performance benchmarks for your code, as both can cause significant and asymmetrical slowdowns.

另一方面，如果您要为代码执行性能基准测试，则不应使用这两个选项中的任何一个，因为两者都会导致明显的且不对称的速度下降。

Note that `--enable-debug` and `--enable-maintainer-zts` change the ABI of the PHP binary, e.g. by adding additional arguments to many functions. As such, shared extensions compiled in debug mode will not be compatible with a PHP binary built in release mode. Similarly a thread-safe extension (ZTS) is not compatible with a non-thread-safe PHP build (NTS).

请注意 `--enable-debug` 和 `--enable-maintainer-zts` 会更改PHP二进制文件的 ABI，例如，通过向许多函数添加额外的参数。 因此，在调试模式下编译的共享扩展将与在发行版的 PHP 二进制文件不兼容。 同样，线程安全扩展（ZTS）与非线程安全 PHP 构建（NTS）不兼容。

Due to the ABI incompatibility `make install` (and PECL install) will put shared extensions in different directories depending on these options:

由于 ABI 不兼容 `make install` （和 PECL 安装）将会根据以下选项将共享扩展放在不同的目录中。

- `$PREFIX/lib/php/extensions/no-debug-non-zts-API_NO` 没有 ZTS 的发行版
- `$PREFIX/lib/php/extensions/debug-non-zts-API_NO` 没有 ZTS 的调试版本
- `$PREFIX/lib/php/extensions/no-debug-zts-API_NO` 带有 ZTS 的发行版本
- `$PREFIX/lib/php/extensions/debug-zts-API_NO` 带有 ZTS 调试版本

The `API_NO` placeholder above refers to the `ZEND_MODULE_API_NO` and is just a date like `20100525`, which is used for internal API versioning.

上面的 `API_NO` 占位符指的是 `ZEND_MODULE_API_NO`，只是一个类似于 `20100525` 的日期，用于内部API版本控制。

For most purposes the configuration switches described above should be sufficient, but of course `./configure` provides many more options, which you’ll find described in the help.

对于大多数目的而言，上述开关的配置就已经足够了，但是 `./configure` 当然可以提供更能多选项，您将在 help 中找到选项的描述。

Apart from passing options to configure, you can also specify a number of environment variables. Some of the more important ones are documented at the end of the configure help output (`./configure --help | tail -25`).

除了传递用于配置的选项外，您还可以指定许多环境变量。一些更重要的信息记录在 configure help 末尾的输出（`./configure --help | tail -25`）。

For example you can use `CC` to use a different compiler and `CFLAGS` to change the used compilation flags:

例如你可以用 `CC` 使用不同的编译器，以及 `CFLAGS` 改变使用的编译标志：
```
~/php-src> ./configure --disable-all CC=clang CFLAGS="-O3 -march=native"
```
In this configuration the build will make use of clang (instead of gcc) and use a very high optimization level (`-O3 -march=native`).

在此配置中，构建将使用 clang（而不是 gcc）并使用非常高的优化级别(`-O3 -march=native`)。

You may use additional compiler warning flags that could help you spot some bugs. For GCC, you may read them in the [GCC manual](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#Warning-Options)

您可以使用其他编译器警告标志，以帮助您发现一些错误。 对于GCC，您可以在GCC手册中阅读它们 [GCC manual](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#Warning-Options)

## <span id="make-and-make-install">make and make install</span>

After everything is configured, you can use `make` to perform the actual compilation:

配置完所有内容后，您可以使用 `make` 来执行实际的编译：
```
~/php-src> make -jN    # where N is the number of cores
```
The main result of this operation will be PHP binaries for the enabled SAPIs (by default `sapi/cli/php` and `sapi/cgi/php-cgi`), as well as shared extensions in the `modules/` directory.

这个操作的主要结果是生成启用了的 SAPI 的 PHP 二进制文件（默认情况下为`sapi/cli/php` 和 `sapi/cgi/php-cgi`），以及 `modules/` 目录中的共享扩展。

Now you can run `make install` to install PHP into `/usr/local` (default) or whatever directory you specified using the `--prefix` configure switch.

现在，您可以运行 `make install` 将PHP安装到 `/usr/local`（默认）或使用`--prefix` 配置开关指定的任何目录中。

`make install` will do little more than copy a number of files to the new location. Unless you specified `--without-pear` during configuration, it will also download and install PEAR. Here is the resulting tree of a default PHP build:

`make install` 所做的只是将大量文件复制到新位置。 除非在配置过程中指定`--without-pear`，否则它将下载并安装 PEAR。这是默认PHP构建的结果树：
```
> tree -L 3 -F ~/myphp

/home/myuser/myphp
|-- bin
|   |-- pear*
|   |-- peardev*
|   |-- pecl*
|   |-- phar -> /home/myuser/myphp/bin/phar.phar*
|   |-- phar.phar*
|   |-- php*
|   |-- php-cgi*
|   |-- php-config*
|   `-- phpize*
|-- etc
|   `-- pear.conf
|-- include
|   `-- php
|       |-- ext/
|       |-- include/
|       |-- main/
|       |-- sapi/
|       |-- TSRM/
|       `-- Zend/
|-- lib
|   `-- php
|       |-- Archive/
|       |-- build/
|       |-- Console/
|       |-- data/
|       |-- doc/
|       |-- OS/
|       |-- PEAR/
|       |-- PEAR5.php
|       |-- pearcmd.php
|       |-- PEAR.php
|       |-- peclcmd.php
|       |-- Structures/
|       |-- System.php
|       |-- test/
|       `-- XML/
`-- php
    `-- man
        `-- man1/
```
A short overview of the directory structure:

目录结构的简短概述：
- _bin/_ 包含了 SAPI 二进制文件 (`php` 和 `php-cgi`), 以及 `phpize` 和 `php-config` 脚本。也是 PEAR/PECL 脚本的所在目录.
- _etc/_ 包含配置文件。注意默认的 php.ini 文件**不在这**。
- _include/php_ 包含头文件，这些头文件是需要构建额外的扩展或者是将 php 键入到自定义软件时所需的。
- _lib/php_ 包含 PEAR 文件。The `lib/php/build` 目录包含构建扩展用的重要的文件，例如，`acinclude.m4` 文件包含 PHP 的 M4 宏。如果我们编译了任何shared 扩展，这些文件将位于 lib/php/extensions 的子目录中。
- _php/man_ 明显包含了针对 `php` 命令的 man 的页面内容。

As already mentioned, the default php.ini location is not etc/. You can display the location using the `--ini` option of the PHP binary:

如前所述，默认的 php.ini 位置不是 etc/。 您可以使用 PHP 二进制文件的 `--ini` 选项显示位置：
```
~/myphp/bin> ./php --ini
Configuration File (php.ini) Path: /home/myuser/myphp/lib
Loaded Configuration File:         (none)
Scan for additional .ini files in: (none)
Additional .ini files parsed:      (none)
```
As you can see the default php.ini directory is `$PREFIX/lib` (libdir) rather than `$PREFIX/etc` (sysconfdir). You can adjust the default php.ini location using the `--with-config-file-path=PATH` configure option.

如您所见，默认的 php.ini 目录为 `$PREFIX/lib`，而不是 `$PREFIX/etc` （sysconfdir）。 您可以使用 `--with-config-file-path=PATH` 配置选项来调整默认的 _php.ini_ 位置。

Also note that `make install` will not create an ini file. If you want to make use of a _php.ini_ file it is your responsibility to create one. For example you could copy the default development configuration:

另请注意 `make install` 不会创建 ini 文件。 如果要使用 _php.ini_ 文件，您应当创建一个文件。 例如，您可以复制默认的开发配置：
```
~/myphp/bin> cp ~/php-src/php.ini-development ~/myphp/lib/php.ini
~/myphp/bin> ./php --ini
Configuration File (php.ini) Path: /home/myuser/myphp/lib
Loaded Configuration File:         /home/myuser/myphp/lib/php.ini
Scan for additional .ini files in: (none)
Additional .ini files parsed:      (none)
```
Apart from the PHP binaries the bin/ directory also contains two important scripts: `phpize` and `php-config`.

除了 PHP 二进制文件，bin/ 目录还包含两个重要的脚本：phpize 和 php-config。

`phpize` is the equivalent of `./buildconf` for extensions. It will copy various files from lib/php/build and invoke autoconf/autoheader. You will learn more about this tool in the next section.

`phpize` 等效于 `./buildconf` 的扩展名。它将从 lib/php/build 复制各种文件并调用 autoconf/autoheader。 您将在下一部分中了解有关此工具的更多信息。

`php-config` provides information about the configuration of the PHP build. Try it out:

`php-config` 提供有关 PHP 构建的配置的信息。 试试看：
```
~/myphp/bin> ./php-config
Usage: ./php-config [OPTION]
Options:
  --prefix            [/home/myuser/myphp]
  --includes          [-I/home/myuser/myphp/include/php -I/home/myuser/myphp/include/php/main -I/home/myuser/myphp/include/php/TSRM -I/home/myuser/myphp/include/php/Zend -I/home/myuser/myphp/include/php/ext -I/home/myuser/myphp/include/php/ext/date/lib]
  --ldflags           [ -L/usr/lib/i386-linux-gnu]
  --libs              [-lcrypt   -lresolv -lcrypt -lrt -lrt -lm -ldl -lnsl  -lxml2 -lxml2 -lxml2 -lcrypt -lxml2 -lxml2 -lxml2 -lcrypt ]
  --extension-dir     [/home/myuser/myphp/lib/php/extensions/debug-zts-20100525]
  --include-dir       [/home/myuser/myphp/include/php]
  --man-dir           [/home/myuser/myphp/php/man]
  --php-binary        [/home/myuser/myphp/bin/php]
  --php-sapis         [ cli cgi]
  --configure-options [--prefix=/home/myuser/myphp --enable-debug --enable-maintainer-zts]
  --version           [5.4.16-dev]
  --vernum            [50416]
```
The script is similar to the `pkg-config` script used by linux distributions. It is invoked during the extension build process to obtain information about compiler options and paths. You can also use it to quickly get information about your build, e.g. your configure options or the default extension directory. This information is also provided by `./php -i` (phpinfo), but `php-config` provides it in a simpler form (which can be easily used by automated tools).

该脚本类似于 linux 发行版中使用的 `pkg-config` 脚本。 在扩展构建过程中调用它以获取有关编译器选项和路径的信息。 您还可以使用它来快速获取有关构建的信息，例如 您的配置选项或默认扩展目录。 `./php -i`（phpinfo）也提供了此信息，但是 `php-config` 以更简单的形式提供了此信息（可以由自动化工具轻松使用）。

## <span id="running-the-test-suit">Running the test suite</span>
If the `make` command finishes successfully, it will print a message encouraging you to run `make test`:

如果 `make` 命令成功完成，它将打印一条消息，鼓励您运行 `make test`：
```
Build complete.
Don't forget to run 'make test'
```
`make test` will run the PHP CLI binary against our test suite, which is located in the different tests/ directories of the PHP source tree. As a default build is run against approximately 9000 tests (less for a minimal build, more if you enable additional extensions) this can take several minutes. The `make test` command is currently not parallel, so specifying the `-jN` option will not make it faster.

`make test` 将针对我们的测试套件运行 PHP CLI 二进制文件，该套件位于 PHP 源代码树的不同 tests/ 目录中。 由于默认构建是针对大约 9000 个测试运行的（对于最小的构建，它会更少，如果启用其他扩展，则更多），这可能需要几分钟。`make test` 命令当前不是并行的，因此指定`-jN`选项不会使其更快。

If this is the first time you compile PHP on your platform, we encourage you to run the test suite. Depending on your OS and your build environment you may find bugs in PHP by running the tests. If there are any failures, the script will ask whether you want to send a report to our QA platform, which will allow contributors to analyze the failures. Note that it is quite normal to have a few failing tests and your build will likely work well as long as you don’t see dozens of failures.

如果这是您第一次在平台上编译 PHP，我们建议您运行测试套件。 根据您的操作系统和构建环境，您可以通过运行测试来发现 PHP 中的错误。 如果有任何故障，脚本将询问您是否要向我们的质量检查平台发送报告，这将使贡献者能够分析故障。请注意，有几次失败的测试是很正常的，只要您看不到数十次失败，您的构建就可能正常运行。

The `make test` command internally invokes the `run-tests.php` file using your CLI binary. You can run `sapi/cli/php run-tests.php --help` to display a list of options this script accepts.

`make test` 命令使用 CLI 二进制文件在内部调用 `run-tests.php` 文件。 您可以运行 `sapi/cli/php run-tests.php --help` 来显示此脚本接受的选项列表。

If you manually run `run-tests.php` you need to specify either the `-p` or `-P` option (or an ugly environment variable):

如果您手动运行 `run-tests.php`，则需要指定 `-p` 或 `-P` 选项（或丑陋的环境变量）：
```
~/php-src> sapi/cli/php run-tests.php -p `pwd`/sapi/cli/php
~/php-src> sapi/cli/php run-tests.php -P
```
`-p` is used to explicitly specify a binary to test. Note that in order to run all tests correctly this should be an absolute path (or otherwise independent of the directory it is called from). `-P` is a shortcut that will use the binary that `run-tests.php` was called with. In the above example both approaches are the same.

`-p` 用于显式指定一个二进制文件来测试。请注意，为了正确运行所有测试，这应该是一个绝对路径（或者独立于从其调用的目录）。`-P` 是一种快捷方式，它将使用与 `run-tests.php` 一起使用的二进制文件。 在上面的示例中，两种方法是相同的。

Instead of running the whole test suite, you can also limit it to certain directories by passing them as arguments to `run-tests.php` .E.g. to test only the Zend engine, the reflection extension and the array functions:

除了运行整个测试套件，您还可以通过将它们作为参数传递给 `run-tests.php` 来将其限制在某些目录中。 例如，仅测试Zend引擎，反射扩展和数组功能：
```
~/php-src> sapi/cli/php run-tests.php -P Zend/ ext/reflection/ ext/standard/tests/array/
```
This is very useful, because it allows you to quickly run only the parts of the test suite that are relevant to your changes. E.g. if you are doing language modifications you likely don’t care about the extension tests and only want to verify that the Zend engine is still working correctly.

这非常有用，因为它允许您仅快速运行测试套件中与更改相关的部分。 例如，如果您要进行语言修改，则可能不关心扩展测试，而只想验证 Zend 引擎是否仍在正常工作。

You don’t need to explicitly use `run-tests.php` to pass options or limit directories. Instead you can use the TESTS variable to pass additional arguments via `make test` .E.g. the equivalent of the previous command would be:

您无需显式使用 `run-tests.php` 来传递选项或限制目录。相反，您可以使用 `TESTS` 变量通过 `make test` 传递其他参数。 与上一个命令等效：
```
~/php-src> make test TESTS="Zend/ ext/reflection/ ext/standard/tests/array/"
```
We will take a more detailed look at the `run-tests.php` system later, in particular also talk about how to write your own tests and how to debug test failures. ==See the dedicated tests chapter.==

稍后，我们将对 `run-tests.php` 系统进行更详细的研究，特别是还讨论如何编写自己的测试以及如何调试测试失败。==查看专用的测试章节==

## <span id="fixing-compilation-problems-and-make-clean">Fixing compilation problems and make clean<span>

As you may know `make` performs an incremental build, i.e. it will not recompile all files, but only those `.c` files that changed since the last invocation. This is a great way to shorten build times, but it doesn’t always work well: For example, if you modify a structure in a header file, `make` will not automatically recompile all `.c` files making use of that header, thus leading to a broken build.

如您所知，make执行增量构建，即它将不会重新编译所有文件，而是仅重新编译自上次调用后的已更改的那些 `.c` 文件。这是缩短构建时间的好方法，但并不总是能很好地工作：例如，如果您修改头文件中的结构，`make` 不会利用该头文件自动重新编译所有 `.c` 文件，从而导致失败的构建。

If you get odd errors while running `make` or the resulting binary is broken (e.g. if `make test` crashes it before it gets to run the first test), you should try to run `make clean`. This will delete all compiled objects, thus forcing the next `make` call to perform a full build.

如果您在运行 `make` 时遇到奇怪的错误，或者生成的二进制文件损坏（例如，如果 `make test` 在运行第一个测试之前将损坏），则应尝试运行 `make clean`。这将删除所有编译的对象，从而在下一个 `make` 命令时执行完整的构建。

Sometimes you also need to run `make clean` after changing `./configure` options. If you only enable additional extensions an incremental build should be safe, but changing other options may require a full rebuild.

有时您还需要在更改 `./configure` 选项后运行 `make clean`。**如果仅启用其他扩展，则增量构建应该是安全的**，但是更改其他选项可能需要完全重建。

A more aggressive cleaning target is available via `make distclean`. This will perform a normal clean, but also roll back any files brought by the `./configure` command invocation. It will delete configure caches, Makefiles, configuration headers and various other files. As the name implies this target “cleans for distribution”, so it is mostly used by release managers.

可通过 `make distclean` 进行更激进的清理。这将执行正常的清理，但还会回滚调用 `./configure` 命令带来的所有文件。它将删除配置缓存，Makefile，配置头和其他各种文件。顾名思义，此目标是“清理分发”，因此它通常由发行的管理者使用。

Another source of compilation issues is the modification of `config.m4` files or other files that are part of the PHP build system. If such a file is changed, it is necessary to rerun the .`/buildconf` script. If you do the modification yourself, you will likely remember to run the command, but if it happens as part of a `git pull` (or some other updating command) the issue might not be so obvious.

编译问题的另一个来源是对 `config.m4` 文件或 PHP 构建系统中的其他文件的修改。如果更改了此类文件，则必须重新运行 `./buildconf` 脚本。 如果您自己进行修改，您可能会记得运行该命令，但是如果它是作为执行 `git pull` （或其他一些更新命令）时产生更改的，则问题可能不会那么明显。

If you encounter any odd compilation problems that are not resolved by `make clean`, chances are that running `./buildconf --force` will fix the issue. To avoid typing out the previous `./configure` options afterwards, you can make use of the `./config.nice` script (which contains your last `./configure` call):

如果遇到了 `make clean` 不能解决的奇怪的编译问题，则运行 `./buildconf --force` 可能会解决该问题。 为避免以后再输入以前的 `./configure` 选项，可以使用 `./config.nice` 脚本（其中包含最后的 `./configure` 调用）：
```
~/php-src> make clean
~/php-src> ./buildconf --force
~/php-src> ./config.nice
~/php-src> make -jN
```
One last cleaning script that PHP provides is `./vcsclean`. This will only work if you checked out the source code from git. It effectively boils down to a call to `g`it clean -X -f -d`, which will remove all untracked files and directories that are ignored by git. You should use this with care.

PHP提供的最后一个清理脚本是 `./vcsclean`。 **仅当您从git中签出源代码时，此方法才有效**。有效地归结为 `git clean -X -f -d` ，这个命令将删除 git 忽略的所有未跟踪文件和目录。 **您应该谨慎使用**。