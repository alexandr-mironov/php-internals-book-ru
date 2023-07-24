« [Введение](/php7/introduction.md) :: [Сборка расширений PHP](/php7/build_system/building_extensions.md) »
___

# Сборка PHP

___

Эта глава объясняет, как Вы можете скомпилировать PHP для разработки расширений или модификаций ядра. Мы будем
рассматривать сборки только для Unix-подобных систем. Если вы хотите собрать PHP в Windows, вам следует взглянуть на
пошаговые инструкции по сборке в вики PHP [[1]](#disclaimer1).

В этой главе также представлен обзор того, как работает система сборки PHP и какие инструменты она использует, но
подробное описание выходит за рамки этой книги.

<a name="#disclaimer1">[1] Отказ от ответственности:</a>

```
Мы не несем ответственности за какие-либо неблагоприятные последствия для здоровья,
вызванные попыткой скомпилировать PHP для Windows.
```

## <a name="#why-not-use-packages">Почему бы не использовать пакеты?</a>

_____
Если Вы уже используете PHP, Вы, вероятно, установили его через диспетчер пакетов, используя команду
например `sudo apt-get install php`. Прежде чем объяснять собственно компиляцию, вы должны сначала понять, почему
Вам необходимо скомпилировать самим и почему нельзя использовать готовую сборку. Для этого есть несколько причин:

Во-первых, предварительно собранный пакет содержит только результирующие двоичные файлы, но упускает другие вещи,
необходимые для компиляции расширений, например заголовочные файлы. Это можно легко исправить, установив пакет
разработки, который обычно называется `php-dev`. Чтобы облегчить отладку с помощью valgrind или gdb, можно
дополнительно установить символы отладки, которые обычно доступны в виде другого пакета под названием `php-dbg`.

Но даже если вы установите заголовки и символы отладки, вы все равно будете работать с релизной сборкой PHP. Это
означает, что он будет построен с высоким уровнем оптимизации, что может сильно затруднить отладку. Кроме того, в
релизных сборках не включены операторы контроля и не создаются предупреждения об утечках памяти. Дополнительно,
предварительно созданные пакеты не обеспечивают безопасность потоков, что может быть полезно для обеспечения сборки
вашего расширения в конфигурации безопасности потоков.

Другая проблема заключается в том, что почти во всех дистрибутивах к PHP применяются дополнительные патчи. В некоторых
случаях эти патчи содержат только незначительные изменения, связанные с конфигурацией, но в некоторых дистрибутивах
используются очень навязчивые патчи, такие как Suhosin. Некоторые из этих патчей несовместимы с низкоуровневыми
расширениями, такими как opcache.

PHP обеспечивает поддержку только для программного обеспечения, которое предоставляется на [php.net](https://php.net),
но не для версий с измененными дистрибутивами. Если вы хотите сообщать об ошибках, отправлять исправления или
использовать наши справочные каналы для написания расширений, вам всегда следует работать с официальной версией "PHP".
Когда мы говорим о «PHP» в этой книге, мы всегда имеем в виду официально поддерживаемую версию.

## <a name="obtaining-the-source-code">Получение исходного кода</a>

_____

Прежде чем вы сможете собрать PHP, вам сперва нужно получить его исходный код. Это можно сделать двумя способами: вы
можете либо загрузить архив со [страницы загрузки PHP](http://www.php.net/downloads.php), либо клонировать и зеркала на
[Github](http://www.github.com/php/php-src).

Процесс сборки немного отличается в обоих случаях: репозиторий git не связывает скрипт `configure`, поэтому вам
нужно сгенерировать его с помощью скрипта `buildconf`, который использует autoconf. Кроме того, репозиторий git не
содержит предварительно сгенерированного лексера и парсера, поэтому вам также потребуется установить re2c и bison.

Мы рекомендуем получать исходный код из git, потому что это простой способ поддерживать исходники обновленными и
опробовать свой код с разными версиями. Также требуется проверка с помощью git, если вы хотите отправлять патчи или пулл
реквесты для PHP.

Чтобы клонировать репозиторий, выполните следующие команды в командной строке:

```shell
~> git clone http://git.php.net/repository/php-src.git
~> cd php-src
# по умолчанию вы будете в ветке master, которая является текущей версией разработки. 
# Вместо этого вы можете переключиться на стабильную ветку
~/php-src> git checkout PHP-8.0
```

Если у вас есть проблемы с `git checkout`, загляните в [Git FAQ](https://wiki.php.net/vcs/gitfaq) на PHP-вики. Git FAQ также объясняет, как
настройть git, если вы хотите внести свой вклад в PHP. Кроме того, он содержит инструкции по настройке нескольких рабочих
каталоги для разных версий PHP. Это может быть очень полезно, если вам нужно проверить свои расширения или изменения на
несколько версий и конфигураций PHP.

Прежде чем продолжить, вам также следует установить некоторые базовые зависимости сборки с помощью вашего менеджера пакетов 
(у вас, вероятно, уже установлены первые три по умолчанию):

* `gcc` и `g++` или какой либо другой набор инструментов компилятора.

* `libc-dev` - предоставляет стандартную библиотеку C, включая файлы заголовков.

* `make` - инструмент управления сборкой используемый в PHP.

* `autoconf` - используется для создания скрипта `configure`.

    * 2.59 или выше (для PHP 7.0-7.1)

    * 2.64 или выше (для PHP 7.2)

    * 2.68 или выше (для PHP 7.3 и выше)

* `libtool` - помогает управлять общими библиотеками.

* `bison` - используется для генерации парсера PHP.

    * 2.4 или выше (для PHP 7.0-7.3)

    * 3.0 или выше (для PHP 7.4 или выше)

* `re2c` - используется для генерации лексера PHP.

    * опционально для PHP <= 7.3.

    * 0.13.4 и выше (для PHP 7.4 и выше)

В Debian/Ubuntu вы можете установить всё это следующей командной:

```shell
~/php-src> sudo apt-get install build-essential autoconf libtool bison re2c
```

В зависимости от расширений которые вы включили на этапе `./configure`, PHP может потребовать определенное количество дополнительных библиотек.
При их установке проверьте, есть ли версия пакета оканчивающаяся на `-dev` или `-devel`, и установите их. 
Пакеты без `dev` обычно не содержат необходимых заголовочных файлов. 
Например, для сборки PHP по умолчанию потребуются `libxml` и `libsqlite3`, которые вы можете установить с помощью пакетов `libxml2-dev` и `libsqlite3-dev`.

Depending on the extensions that you enable during the `./configure` stage PHP will need a number of additional
libraries. When installing these, check if there is a version of the package ending in `-dev` or `-devel` and
install them instead. The packages without `dev` typically do not contain necessary header files. For example a
default PHP build will require libxml and libsqlite3, which you can install via the `libxml2-dev`
and `libsqlite3-dev` packages.

## <a name="#build-overview">Обзор процесса сборки</a>

_____

Прежде чем более более детально рассмотреть что делают отдельные шаги сборки, команды ниже используются для сборки PHP «по умолчанию»:

Before taking a closer look at what the individual build steps do, here are the commands you need to execute for a
“default” PHP build:

```shell
~/php-src> ./buildconf # only necessary if building from git
~/php-src> ./configure
~/php-src> make -jN 
```

Для быстрой сборки замените N на количество ядер CPU которое Вам доступно (можно определить запустив `nproc`) 

For a fast build, replace N with the number of CPU cores you have available (you can run `nproc` to determine this).

By default PHP will build binaries for the CLI and CGI SAPIs, which will be located at `sapi/cli/php`
and `sapi/cgi/php-cgi`
respectively. To check that everything went well, try running `sapi/cli/php -v`.

Additionally you can run `sudo make install` to install PHP into `/usr/local`. The target directory can be
changed by specifying a `--prefix` in the configuration stage:

```shell
~/php-src> ./configure --prefix=$HOME/myphp
~/php-src> make -jN
~/php-src> make install
```

Here `$HOME/myphp` is the installation location that will be used during the `make install` step. Note that
installing PHP is not necessary, but can be convenient if you want to use your PHP build outside of extension
development.

Now lets take a closer look at the individual build steps!

## <a name="#the-buildconf-script">The ./buildconf script</a>

_____

If you are building from the git repository, the first thing you’ll have to do is run the `./buildconf` script. This
script does little more than invoking the `build/build.mk` makefile, which in turn calls `build/build2.mk`.

The main job of these makefiles is to run `autoconf` to generate the `./configure` script and `autoheader`
to generate the `main/php_config.h.in` template. 
The latter file will be used by configure to generate the final configuration header file `main/php_config.h`.

Both utilities produce their results from the configure.ac file (which specifies most of the PHP build process), the
`build/php.m4` file (which specifies a large number of PHP-specific M4 macros) and the config.m4 files of individual
extensions and SAPIs (as well as a bunch of other [m4 files](http://www.gnu.org/software/m4/m4.html)).

The good news is that writing extensions or even doing core modifications will not require much interaction with the
build system. You will have to write small `config.m4` files later on, but those usually just use two or three of
the high-level macros that `build/php.m4` provides. As such we will not go into further detail here.

The `./buildconf` script only has two options: `--debug` will disable warning suppression when calling autoconf
and autoheader. Unless you want to work on the buildsystem, this option will be of little interest to you.

The second option is `--force`, which will allow running `./buildconf` in release packages (e.g. if you
downloaded the packaged source code and want to generate a new `./configure`) and additionally clear the
configuration caches `config.cache` and `autom4te.cache/`.

If you update your git repository using git pull (or some other command) and get weird errors during the make step, this
usually means that something in the build configuration changed and you need to rerun ./buildconf.

## <a name="#the-configure-script">The ./configure script</a>

_____

Once the `./configure` script is generated you can make use of it to customize your PHP build. You can list all
supported options using `--help`:

```shell
~/php-src> ./configure --help | less 
```

The first part of the help will list various generic options, which are supported by all autoconf-based configuration
scripts. One of them is the already mentioned `--prefix=DIR`, which changes the installation directory used by make
install. Another useful option is `-C`, which will cache the result of various tests in the config.cache file and speed up
subsequent `./configure` calls. Using this option only makes sense once you already have a working build and want to
quickly change between different configurations.

Apart from generic autoconf options there are also many settings specific to PHP. For example, you can choose which
extensions and SAPIs should be compiled using the `--enable-NAME` and `--disable-NAME` switches. If the extension or SAPI
has external dependencies you need to use --with-NAME and --without-NAME instead.

If a library needed by NAME is not located in the default location (e.g. because you compiled it yourself), some
extensions allow you to specify its location using `--with-NAME=DIR`. However, since PHP 7.4 most extensions use
pkg-config instead, in which case passing a directory to --with has no effect. In this case, it is necessary to add the
library to the PKG_CONFIG_PATH:

```shell
export PKG_CONFIG_PATH=/path/to/library/lib/pkgconfig:$PKG_CONFIG_PATH 
```

By default PHP will build the CLI and CGI SAPIs, as well as a number of extensions. You can find out which extensions
your PHP binary contains using the -m option. For a default PHP 7.0 build the result will look as follows:

```shell
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

If you now wanted to stop compiling the CGI SAPI, as well as the `tokenizer` and `sqlite3` extensions and instead enable
`opcache` and `gmp`, the corresponding configure command would be:

```shell
~/php-src> ./configure --disable-cgi --disable-tokenizer --without-sqlite3 \
--enable-opcache --with-gmp 
```

By default most extensions will be compiled statically, i.e. they will be part of the resulting binary. Only the opcache
extension is shared by default, i.e. it will generate an `opcache.so` shared object in the `modules/` directory. You can
compile other extensions into shared objects as well by writing `--enable-NAME=shared` or `--with-NAME=shared` (but not all
extensions support this). We’ll talk about how to make use of shared extensions in the next section.

To find out which switch you need to use and whether an extension is enabled by default, check `./configure --help`. If
the switch is either `--enable-NAME` or `--with-NAME` it means that the extension is not compiled by default and needs
to be explicitly enabled. `--disable-NAME` or `--without-NAME` on the other hand indicate an extension that is compiled by
default, but can be explicitly disabled.

Some extensions are always compiled and can not be disabled. To create a build that only contains the minimal amount of
extensions use the `--disable-all` option:

```shell
~/php-src> ./configure --disable-all && make -jN
~/php-src> sapi/cli/php -m
[PHP Modules]
Core date hash json pcre Reflection SPL standard
``` 

The `--disable-all` option is very useful if you want a fast build and don’t need much functionality (e.g. when
implementing language changes). For the smallest possible build you can additionally specify the `--disable-cgi`
switch, so only the CLI binary is generated.

There are three more switches, which you should usually specify when developing extensions or working on PHP:

`--enable-debug` enables debug mode, which has multiple effects: Compilation will run with `-g` to generate debug
symbols and additionally use the lowest optimization level -O0. This will make PHP a lot slower, but make debugging with
tools like gdb more predictable. Furthermore debug mode defines the `ZEND_DEBUG` macro, which will enable th use of
assertions and enable various debugging helpers in the engine. Among other things memory leaks, as well as incorrect use
of some data structures, will be reported. It is possible to enable debug assertions without disabling optimizations by
using
`--enable-debug-assertions` instead.

`--enable-zts` (or `--enable-maintainer-zts` before PHP 8.0) enables thread-safety. This switch will define the
ZTS macro, which in turn will enable the whole TSRM (thread-safe resource manager) machinery used by PHP. Since PHP 7
having this switch continuously enabled is much less important than on previous versions. It is primarily important to
make sure you included all the necessary boilerplate code. If you need more information about thread safety and global
memory management in PHP, you should read the globals management chapter

`--enable-werror` (since PHP 7.4) enables the `-Werror` compiler flag, which will promote compiler warnings to
errors. Enabling this flag ensures that the PHP build remains warning free. However, generated warnings depend on the
used compiler, version and optimizatino options, so some compilers may not be usable with option.

On the other hand you should not use the --enable-debug option if you want to perform performance benchmarks for your
code. --enable-zts can also negatively impact runtime performance.

Note that `--enable-debug` and `--enable-zts` change the ABI of the PHP binary, e.g. by adding additional
arguments to functions. As such, shared extensions compiled in debug mode will not be compatible with a PHP binary built
in release mode. Similarly a thread-safe extension (ZTS) is not compatible with a non-thread-safe PHP build (NTS).

Due to the ABI incompatibility make install (and PECL install) will put shared extensions in different directories
depending on these options:

* `$PREFIX/lib/php/extensions/no-debug-non-zts-API_NO` for release builds without ZTS
* `$PREFIX/lib/php/extensions/debug-non-zts-API_NO` for debug builds without ZTS
* `$PREFIX/lib/php/extensions/no-debug-zts-API_NO` for release builds with ZTS
* `$PREFIX/lib/php/extensions/debug-zts-API_NO` for debug builds with ZTS

The API_NO placeholder above refers to the ZEND_MODULE_API_NO and is just a date like 20100525, which is used for
internal API versioning.

For most purposes the configuration switches described above should be sufficient, but of course ./configure provides
many more options, which you’ll find described in the help.

Apart from passing options to configure, you can also specify a number of environment variables. Some of the more
important ones are documented at the end of the configure help output (`./configure --help | tail -25`).

For example you can use CC to use a different compiler and `CFLAGS` to change the used compilation flags:

```shell
~/php-src> ./configure --disable-all CC=clang CFLAGS="-O3 -march=native"
```

In this configuration the build will make use of clang (instead of gcc) and use a very high optimization level (`-O3
-march=native`).

An option that is particularly useful for development is `-fsanitize`, which allows you to detect memory corruption and
undefined behavior at runtime:

```shell
CFLAGS="-fsanitize=address -fsanitize=undefined"
```

These options only work reliably since PHP 7.4 and will significantly slow down the generated PHP binary.

## <a name="#make-and-make-install">make and make install</a>

_____
After everything is configured, you can use make to perform the actual compilation:

```bash
~/php-src> make -jN # where N is the number of cores
```
The main result of this operation will be PHP binaries for the enabled SAPIs (by default `sapi/cli/php` and `sapi/cgi/php-cgi`), as well as shared extensions in the `modules/` directory.

Now you can run make install to install PHP into /usr/local (default) or whatever directory you specified using the
`--prefix configure switch`.

make install will do little more than copy a number of files to the new location. If you specified `--with-pear` during
configuration, it will also download and install PEAR. Here is the resulting tree of a default PHP build:

```shell
> tree -L 3 -F ~/myphp

/home/myuser/myphp |-- bin | |-- pear*
| |-- peardev*
| |-- pecl*
| |-- phar -> /home/myuser/myphp/bin/phar.phar*
| |-- phar.phar*
| |-- php*
| |-- php-cgi*
| |-- php-config*
|   `-- phpize*
|-- etc |   `-- pear.conf |-- include
|   `-- php | |-- ext/ | |-- include/ | |-- main/ | |-- sapi/ | |-- TSRM/ |       `-- Zend/ |-- lib
|   `-- php | |-- Archive/ | |-- build/ | |-- Console/ | |-- data/ | |-- doc/ | |-- OS/ | |-- PEAR/ | |-- PEAR5.php | |-- pearcmd.php | |-- PEAR.php | |-- peclcmd.php | |-- Structures/ | |-- System.php | |-- test/ |       `
-- XML/
`-- php
`-- man
`-- man1/ 
```

A short overview of the directory structure:

- `bin/` contains the SAPI binaries (`php` and `php-cgi`), as well as the `phpize` and `php-config` scripts. It is also home to the
various PEAR/PECL scripts.

- `etc/` contains configuration. Note that the default `php.ini` directory is not here.

- `include/php` contains header files, which are needed to build additional extensions or embed PHP in custom software.

- `lib/php` contains PEAR files. The `lib/php/build` directory includes files necessary for building extensions, e.g. the
`php.m4` file containing PHP’s M4 macros. If we had compiled any shared extensions those files would live in a
subdirectory of `lib/php/extensions`.

- `php/man` obviously contains man pages for the php command.

As already mentioned, the default php.ini location is not etc/. You can display the location using the --ini option of
the PHP binary:

```bash
~/myphp/bin> ./php --ini
Configuration File (php.ini) Path: /home/myuser/myphp/lib
Loaded Configuration File:         (none)
Scan for additional .ini files in: (none)
Additional .ini files parsed:      (none)
```

As you can see the default php.ini directory is $PREFIX/lib (libdir) rather than `$PREFIX/etc` (sysconfdir). You can
adjust the default php.ini location using the `--with-config-file-path=PATH` configure option.

Also note that make install will not create an ini file. If you want to make use of a `php.ini` file it is your
responsibility to create one. For example you could copy the default development configuration:

```bash
~/myphp/bin> cp ~/php-src/php.ini-development ~/myphp/lib/php.ini
~/myphp/bin> ./php --ini
Configuration File (php.ini)
Path: /home/myuser/myphp/lib
Loaded Configuration File: /home/myuser/myphp/lib/php.ini
Scan for additional .ini files in: (none)
Additional .ini files parsed:      (none)
```
Apart from the PHP binaries the bin/ directory also contains two important scripts: `phpize` and `php-config`.

`phpize` is the equivalent of `./buildconf` for extensions. It will copy various files from `lib/php/build` and invoke
`autoconf/autoheader`. You will learn more about this tool in the next section.

`php-config` provides information about the configuration of the PHP build. Try it out:

```bash
~/myphp/bin> ./php-config Usage: ./php-config [OPTION]
Options:
--prefix            [/home/myuser/myphp]
--includes          [-I/home/myuser/myphp/include/php -I/home/myuser/myphp/include/php/main -I/home/myuser/myphp/include/php/TSRM -I/home/myuser/myphp/include/php/Zend -I/home/myuser/myphp/include/php/ext -I/home/myuser/myphp/include/php/ext/date/lib]
--ldflags           [ -L/usr/lib/i386-linux-gnu]
--libs              [-lcrypt -lresolv -lcrypt -lrt -lrt -lm -ldl -lnsl -lxml2 -lxml2 -lxml2 -lcrypt -lxml2 -lxml2 -lxml2 -lcrypt ]
--extension-dir     [/home/myuser/myphp/lib/php/extensions/debug-zts-20100525]
--include-dir       [/home/myuser/myphp/include/php]
--man-dir           [/home/myuser/myphp/php/man]
--php-binary        [/home/myuser/myphp/bin/php]
--php-sapis         [ cli cgi]
--configure-options [--prefix=/home/myuser/myphp --enable-debug --enable-maintainer-zts]
--version           [5.4.16-dev]
--vernum            [50416]
```
The script is similar to the pkg-config script used by linux distributions. It is invoked during the extension build
process to obtain information about compiler options and paths. You can also use it to quickly get information about
your build, e.g. your configure options or the default extension directory. This information is also provided by ./php
-i (phpinfo), but php-config provides it in a simpler form (which can be easily used by automated tools).

Running the test suite If the make command finishes successfully, it will print a message encouraging you to run make
test:

Build complete. Don't forget to run 'make test' make test will run the PHP CLI binary against our test suite, which is
located in the different tests/ directories of the PHP source tree. As a default build is run against more than 10000 (
less for a minimal build, more if you enable additional extensions) this can take several minutes.

The make test command internally invokes the run-tests.php file using your CLI binary. For more control, it is
recommended to invoke run-tests.php directly. For example, this will allow you to enable the parallel test runner:

```bash
~/php-src> sapi/cli/php run-tests.php -jN
```
Test parallelism is only available as of PHP 7.4. On earlier PHP versions parallelism is not available, 
and it is necessary to additionally pass the `-P` option:

```bash
~/php-src> sapi/cli/php run-tests.php -P
```
Instead of running the whole test suite, you can also limit it to certain directories by passing them as arguments to run-tests.php. 
E.g. to test only the Zend engine, the reflection extension and the array functions:

```bash
~/php-src> sapi/cli/php run-tests.php -jN Zend/ ext/reflection/ ext/standard/tests/array/
```
This is very useful, because it allows you to quickly run only the parts of the test suite that are relevant to your changes. 
E.g. if you are doing language modifications you likely don’t care about the extension tests 
and only want to verify that the Zend engine is still working correctly.

You can run `sapi/cli/php run-tests.php --help` to display a full list of options the test runner accepts. 
Some particularly useful options are:

- `-c php.ini` can be used to specify a `php.ini` file to use.

- `-d foo=bar` can be used to set ini options.

- `-m` runs tests under valgrind to detect memory errors. Note that this is extremely slow.

- `--asan` should be set when compiling PHP with `-fsanitize=address`. Together these are approximately equivalent to running
under valgrind, but with much better performance.

You don’t need to explicitly use `run-tests.php` to pass options or limit directories. Instead you can use the TESTS
variable to pass additional arguments via make test. E.g. the equivalent of the previous command would be:

```bash
~/php-src> make test TESTS="-jN Zend/ ext/reflection/ ext/standard/tests/array/"
```
We will take a more detailed look at the run-tests.php system later, in particular also talk about how to write your own
tests and how to debug test failures. See the dedicated tests chapter.

### Fixing compilation problems and `make clean` 

As you may know `make` performs an incremental build, i.e. it will not recompile all files, 
but only those `.c` files that changed since the last invocation. 
This is a great way to shorten build times, but it doesn’t always work well: 
For example, if you modify a structure in a header file, make will not
automatically recompile all `.c` files making use of that header, thus leading to a broken build.

If you get odd errors while running `make` or the resulting binary is broken (e.g. if `make test` crashes it before it gets
to run the first test), you should try to run `make clean`. This will delete all compiled objects, thus forcing the next
`make` call to perform a full build. (You can use `ccache` to reduce the cost of rebuilds.)

Sometimes you also need to run `make clean` after changing `./configure` options. If you only enable additional extensions
an incremental build should be safe, but changing other options may require a full rebuild.

Another source of compilation issues is the modification of config.m4 files or other files that are part of the PHP
build system. If such a file is changed, it is necessary to rerun the `./buildconf` and `./configure` scripts. If you do the
modification yourself, you will likely remember to run the command, but if it happens as part of a git pull (or some
other updating command) the issue might not be so obvious.

If you encounter any odd compilation problems that are not resolved by make clean, chances are that running `./buildconf`
will fix the issue. To avoid typing out the previous `./configure` options afterwards, you can make use of the
`./config.nice` script (which contains your last `./configure` call):

```bash
~/php-src> make clean
~/php-src> ./buildconf --force
~/php-src> ./config.nice
~/php-src> make -jN
```
One last cleaning script that PHP provides is `./vcsclean`. This will only work if you checked out the
source code from git. It effectively boils down to a call to `git clean -X -f -d`, which will remove all untracked files
and directories that are ignored by git. You should use this with care.

