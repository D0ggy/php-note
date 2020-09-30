# Building PHP
-----------------
This chapter explains how you can compile PHP in a way that is suitable for development of extensions or core modifications. We will only cover builds on Unixoid systems. If you wish to build PHP on Windows, you should take a look at the step-by-step build instructions in the PHP wiki [1].

本章解释你怎样能以适合开发扩展和内核修改的方式来编译 PHP。我们将只介绍 Unixoid 系统上的构建。如果你想要在 Windows 上构建 PHP, 你应该看一下 [step-by-step build instructions](https://wiki.php.net/internals/windows/stepbystepbuild_sdk_2)，[[1]](#id3)。

This chapter also provides an overview of how the PHP build system works and which tools it uses, but a detailed description is outside the scope of this book.

本章概述了 PHP 构建系统的工作方式和使用的工具，详细的描述不在本书讨论范围内。

<span id="id3">[1]</span> 免责声明: 对在 Windows 上编译 PHP 产生的后果概不负责。

## Why not use packages?
---------------------------
If you are currently using PHP, you likely installed it through your package manager, using a command like `sudo apt-get install php`. Before explaining the actual compilation you should first understand why doing your own compile is necessary and you can’t just use a prebuilt package. There are multiple reasons for this:

如果你当前正使用 PHP，以可能通过包管理器，类似 `sudo apt-get install php` 的命令安装的。在解释实际编译之前你应该首先明白为什么自己编译是重要的并且你不能只用预编译的包。有多个原因：

Firstly, the prebuilt package only contains the resulting binaries, but misses other things that are necessary to compile extensions, e.g. header files. This can be easily remedied by installing a development package, which is typically called `php-dev`. To facilitate debugging with valgrind or gdb one could additionally install debug symbols, which are usually available as another package called `php-dbg`.

首先，预编译包仅仅包含生成的二进制文件，错过了其他的重要的编译扩展的内容，例如头文件。可以通过安装开发包轻松解决，通常被称为 `php-dev`。为了便于使用 valgrind 或 gdb 进行调试，可以另外安装调试符号，这些符号通常作为另一个名为 `php-dbg` 的软件包提供。

But even if you install headers and debug symbols, you’ll still be working with a release build of PHP. This means that it will be built with high optimization level, which can make debugging very hard. Furthermore release builds will not generate warnings about memory leaks or inconsistent data structures. Additionally prebuilt packages don’t enable thread safety, which is very helpful during development.

但是即使你安装了头文件和调试符号，你仍将使用 PHP 的发行版。意味着将以最高优化等级构建 PHP，使得调试非常困难。此外，发行版本不会生成有关内存泄漏或数据结构不一致的警告。另外，预编译版本不能启用线程安全，这在开发中很有帮助。

Another issue is that nearly all distributions apply additional patches to PHP. In some cases these patches only contain minor changes related to configuration, but some distributions make use of highly intrusive patches like Suhosin. Some of these patches are known to introduce incompatibilities with low-level extensions like opcache.

另外一个问题是，几乎所有发行版本都为 PHP 应用了额外的补丁。在某些情况下，这些补丁仅包含了与配置相关的微小更改，但是某些发行版使用了 Suhosin 等高度侵入性的修补程序。已知一些补丁被介绍与低级扩展（例如 opcache）不兼容。

PHP only provides support for the software as provided on php.net and not for the distribution-modified versions. If you want to report bugs, submit patches or make use of our help channels for extension-writing, you should always work against the official PHP version. When we talk about “PHP” in this book, we’re always referring to the officially supported version.

PHP 仅对 [php.net](https://php.net/) 上提供的软件提供支持，而对发行版修改的版本不提供支持。如果您想报告错误，提交补丁或利用我们的帮助渠道进行扩展编写，则应始终对照正式的PHP版本进行工作。当我们在本书中谈论 “PHP” 时，我们总是指受官方支持的版本。

## Obtaining the source code
---------------------------
Before you can build PHP you first need to obtain its source code. There are two ways to do this: You can either download an archive from PHP’s download page or clone the git repository from git.php.net (or the mirror on Github).

在构建 PHP 前，你首先需要获取它的源码。这里有两种方法：你可以从 [PHP’s download page](https://php.net/downloads.php) 下载一个归档文件，或者从 [git.php.net](http://git.php.net/) 克隆 git 仓库（或者从 GitHub 的镜像）。

The build process is slightly different for both cases: The git repository doesn’t bundle a configure script, so you’ll need to generate it using the buildconf script, which makes use of autoconf. Furthermore the git repository does not contain a pregenerated parser, so you’ll also need to have bison installed.

两种构建过程有略微不同：git 仓库中没有捆绑一个 `configure` 脚本，所以你要用 `autoconf` 生成。此外，git 仓库中不包含预生成解析器，所以你也需要安装 bison。 

We recommend to checkout out the source code from git, because this will provide you with an easy way to keep your installation updated and to try your code with different versions. A git checkout is also required if you want to submit patches or pull requests for PHP.

我们推荐从 git 检出源码，因为这会使安装、更新和尝试不同的版本更简单。如果你想为 PHP 贡献代码或者拉取 PHP ，那么从 git 检出还是有必要的。

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

如果你使用 git checkout 有问题，在 PHP wiki 浏览一下 [Git FAQ](https://wiki.php.net/vcs/gitfaq)。如果你想为 PHP 贡献，Git FAQ 还解释了怎设置 git。另外，它包含了关于设置不同版本的 PHP 的数明。如果你想针对多个 PHP 版本和配置测试扩展活更改，这也很有用。

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

在 Debian/Ubuntu 你可以通过以下命令安装所有这些：

```shell
~/php-src> sudo apt-get install build-essential autoconf libtool bison re2c
```
Depending on the extensions that you enable during the `./configure` stage PHP will need a number of additional libraries. When installing these, check if there is a version of the package ending in `-dev` or `-devel` and install them instead. The packages without `dev` typically do not contain necessary header files. For example a default PHP build will require libxml, which you can install via the `libxml2-dev` package.

根据你启用的这些扩展，在 `./configure` 阶段 PHP 需要许多其他库。当安装这些时，请检查是否有以 `-dev` 或 `-devel` 结尾的软件包版本，然后安装它们。没有 `dev` 的软件通常不包含必要的头文件。例如，默认的 PHP 构建将需要 `libxml`，你可以安装 `libxml2-dev` 的包。

If you are using Debian or Ubuntu you can use `sudo apt-get build-dep php7` to install a large number of optional build-dependencies in one go. If you are only aiming for a default build, many of them will not be necessary though.

如果您使用的是 Debian 或 Ubuntu，你可以用 `sudo apt-get build-dep php7` 一次性安装大量的可选的构建依赖项。如果您仅针对默认构建，则其中许多将不是必须的。

## Build overview 构建概览
---------------------------

Before taking a closer look at what the individual build steps do, here are the commands you need to execute for a “default” PHP build:

再仔细研究各个步骤之前，这是你需要为 "defalt" PHP 的构建执行的命令：

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

如果你将从 git 仓库构建，第一件事是你必须运行 `./buildconf` 脚本。这个脚本不止调用 `build/build.mk` makefile，其他的还调用 `build/build2.mk`。

The main job of these makefiles is to run `autoconf` to generate the `./configure` script and `autoheader` to generate the `main/php_config.h.in` template. The latter file will be used by configure to generate the final configuration header file `main/php_config.h`.



Both utilities produce their results from the `configure.in` file (which specifies most of the PHP build process), the `acinclude.m4` file (which specifies a large number of PHP-specific M4 macros) and the `config.m4` files of individual extensions and SAPIs (as well as a bunch of other m4 files).

The good news is that writing extensions or even doing core modifications will not require much interaction with the build system. You will have to write small `config.m4` files later on, but those usually just use two or three of the high-level macros that `acinclude.m4` provides. As such we will not go into further detail here.

The `./buildconf` script only has two options: `--debug` will disable warning suppression when calling autoconf and autoheader. Unless you want to work on the buildsystem, this option will be of little interest to you.

The second option is `--force`, which will allow running `./buildconf` in release packages (e.g. if you downloaded the packaged source code and want to generate a new `./configure`) and additionally clear the configuration caches `config.cache` and `autom4te.cache/`.

If you update your git repository using `git pull` (or some other command) and get weird errors during the `make` step, this usually means that something in the build configuration changed and you need to run `./buildconf --force`.