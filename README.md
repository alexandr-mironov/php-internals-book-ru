# php-internals-book-ru
PHP internals book

Содержимое

PHP7 и PHP8
____
* [Введение](/php7/introduction.md)
* [Использование системы сборки PHP](/php7/build_system.md)
  * [Сборка PHP](/php7/build_system/build_php.md)
    * [Почему не использовать пакеты?](/php7/build_system/build_php.md#why-not-use-packages)
    * [Obtaining the source code](/php7/build_system/build_php.md#obtaining-the-source-code)
    * [Build overview](/php7/build_system/build_php.md#build-overview)
    * The ./buildconf script
    * The ./configure script
    * make and make install
    * Running the test suite
    * Fixing compilation problems and make clean
  * Building PHP extensions
    * Loading shared extensions
    * Installing extensions from PECL
    * Adding extensions to the PHP source tree
    * Building extensions using phpize
    * Displaying information about extensions
* Zvals
  * Basic structure
    * Types and values
    * The zval struct
    * Access macros
  * Memory management
    * Reference-counted values
    * Zval memory management
  * References
    * Reference semantics
    * Representation
    * Initializing references
    * Dereferencing and unwrapping
    * Indirect zvals
  * Casts and operations
    * Casts
    * Operations
* Internal types
  * Strings
    * The zend_string API
    * smart_str API
    * PHP’s custom printf functions
  * The Resource type: zend_resource
    * What is the “Resource” type?
    * Resource types and resource destruction
    * Playing with resources
    * Reference counting resources
    * Persistent resources
  * HashTables: zend_array
  * Functions: zend_function
  * Objects and classes
* Extensions design
  * Learning the PHP lifecycle
    * The parallelism models
    * The PHP extensions hooks
    * Hooking by overwriting function pointers
  * A look into a PHP extension and extension skeleton
    * How the engine loads extensions
    * What is a PHP extension ?
    * Generating extension skeleton with scripts
    * New age of the extension skeleton generator
    * Publishing API
  * Registering and using PHP functions
    * zend_function_entry structure
    * Registering PHP functions
    * Declaring function arguments
    * The PHP function structure and API, in C
    * Adding tests
    * Playing with constants
    * A go with Hashtables (PHP arrays)
    * Managing references
  * Managing global state
    * Managing request globals
    * Using true globals
  * Publishing extension information
    * MINFO() hook
    * A note about the Reflection API
  * Hooks provided by PHP
    * Hooking into the execution of functions
    * Overwriting an Internal Function
    * Modifying the Abstract Syntax Tree (AST)
    * Hooking into Script/File Compilation
    * Notification when Error Handler is called
    * Notification when Exception is thrown
    * Hooking into eval()
    * Hooking into the Garbage Collector
    * Overwrite Interrupt Handler
    * Replacing Opcode Handlers
  * Declaring and using INI settings
    * Reminders on INI settings
    * Zoom on INI settings
    * Registering and using INI entries
    * Validators and globals memory bridge
    * Using a displayer
  * Zend Extensions
    * On differences between PHP and Zend extensions
    * What is a Zend extension ?
    * Why need a Zend extension ?
    * API versions and conflicts management
    * Zend extensions lifetime hooks
    * Practice : my first example Zend extension
    * Hybrid extensions
* Memory management
  * Zend Memory Manager
    * The two main kind of dynamic memory pools in PHP
    * Zend Memory Manager API
    * Zend Memory Manager debugging shields
    * ZendMM internal design
    * Common errors and mistakes
  * Debugging memory
    * A quick note about valgrind
    * Before starting
    * Memory leak detection example
    * Buffer overflow/underflow detection
    * Conclusions
* Zend engine
  * Zend Compiler
  * Zend Executor
  * Zend OPCache
* Debugging with GDB