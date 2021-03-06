
Начнем с малого, как прикрутить `Android NDK` к `C::B`

Есть два пути:

- 1. прописать на каждую платформу свой профиль, с исполняемыми файлами согласно платформе. На мой взгляд утомительно, много всего и непонятно зачем. С учетом что если собирать одно и тоже под все платформы, то объем телодвижений с переключением конфигураций довольно велик.  
У меня по умолчанию собираются `arm64-v8a, armeabi-v7a, x86, x86_64`.  
- 2. использовать родную систему сборки `ndk-build`. Это самое простое и изящное решение, не вступающее в конфликты с большинством проверок `C::B`, но, в настройках `C::B` важна каждая деталь, схема довольно капризная, и при неточностях может легко ломаться.  

### Возможности решения с помощью ndk-build:

- не требуется рутированного устройства  
- полноценная отладка сборки, без дополнительных инструментов (набор инструментов NDK)  
- полноценная сборка в `Debug/Release` режимах  
- полноценный запуск приложения (с устройства)  
- авто запуск/остановка `gdbserver` с устройства  
- не требуется оберток из `Gradle/Java` кода, работает напрямую с устройством  
- полная интеграция проектов `C::B` при использвании утилиты экспорта [cpb2ndk](CBP2NDK.RU.md).  

### Интеграция Android NDK

----------

- __Путь перый__: скачать и установить готовый инсталлятор `NDK C::B template`, [подробнее..](CBNDKAUTOINSTALL.RU.md)  
- __Путь второй__: в ручном режиме устанавливать `NDK C::B template` для каждого проекта, как это сделать описано ниже.  

----------

Идеология и манера сборки приложения с использованием тулчейна `Android NDK` максимально приближена к типовому поведению `C::B`, логика процессов:  

- сборка в режиме `Release`- этапы: всегда новая сборка приложения, копирование его на устройство, запуск приложения. Все действия отображаются в консоле `C::B`.  
- сборка в режиме `Debug`- этапы: всегда новая сборка приложения, копирование его на устройство, копирование `NDK-парт` файлов для отладки `gdbserver`, `gdb.setup`, запуск `gdbserver` на устройстве, ожидание подключения дебагера `GDB` для отладки.  
- режим `запуска приложения`- этапы: запуск на устройстве приложения с выводом результатов в консоль.  
- режим `Отладка -> Старт` - этапы: всегда новая сборка приложения, копирование его на устройство, копирование `NDK-парт`файлов для отладки `gdbserv`, `gdb.setup`, запуск `gdbserv` на устройстве, автоматическое подключение дебагера `GDB`и переход в режим отладки.  
- в режимах `Debug`, `Отладка -> Старт`, окно запускаемого `gdbserver` стартует в минимизированном состоянии и автоматически закрывается по окончани отладки.


### Файлы шаблона:

Сам проект `C::B`, важные секции:

- `<Build><Option output>` - указывает на скрипт дистанционного запуска приложения на устройстве - `RunRemote.cmd`. Скрипт генерируется автоматически.  
- `<Build><Option compiler="android_ndk-build">` - это обработанное название компилятора в настройках `C::B` - `Android NDK-Build`. Как завести учетную запись нового компилятора, показано ниже.

В секции `<Extensions><debugger><remote_debugging>` находятся настройки дистанционной отладки, реализованной с помощью `GDB`:

- `options ip_address="127.0.0.1" ip_port="59999" extended_remote="0"`, если есть необходимость изменить номер порта, то это так-же необходимо сделать в `Makefile`. Если опция `extended_remote` будет отличной от нуля, окно дистанционно запущенного `gdbserver` не закроется автоматически после отладки.

[**AndroidNdkTemplate.cbp**](https://github.com/ClnViewer/Code-Blocks-Android-NDK/blob/master/CB-Template/AndroidNdkTemplate.cbp) - файл проекта `C::B`.  

> Если в процессе запуска собранного приложения на устройстве у вас возникают следующие ошибки:

    WARNING: linker: /data/local/tmp/youprog: unused DT entry: type 0x6ffffef5 arg 0x4040
    WARNING: linker: /data/local/tmp/youprog: unused DT entry: type 0x6ffffffe arg 0x5a34
    WARNING: linker: /data/local/tmp/youprog: unused DT entry: type 0x6fffffff arg 0x2

> Используйте [android-elf-cleaner](https://github.com/ClnViewer/Code-Blocks-Android-NDK/android-elf-cleaner)   
> Эти ошибки характерны для бинарного файла платформы `armeabi-v7a`, позднее они были исправлены на последних версиях библиотек.  

Скачать [android-elf-cleaner v.0.0.14.79/win32 (03.07.2019)](https://clclon.github.io/Code-Blocks-Android-NDK/android-elf-cleaner.zip) 

### Исходные файлы управления сборкой в директории NDK проекта:

----------

[**Application.mk**](https://github.com/ClnViewer/Code-Blocks-Android-NDK/blob/master/CB-Template/Application.mk) - установки параметров сборки.  
[**Android.mk**](https://github.com/ClnViewer/Code-Blocks-Android-NDK/blob/master/CB-Template/Android.mk) - собственно и является мейк-файлом уникальным для каждого `NDK` проекта (приложения).  
[**Makefile**](https://github.com/ClnViewer/Code-Blocks-Android-NDK/blob/master/CB-Template/Makefile) - непосредственно запускается `C::B`:

Файлы `Application.mk` и `Makefile` являются универсальными для всех проектов собираемых с помощью `NDK` и не требуют правок.

Для понимания структуры где чего лежит в проекте `NDK` приложения, приведу дерево:

    │   Android.mk
    │   AndroidNdkTemplate.cbp
    │   Application.mk
    │   main.c
    │   Makefile
    │   
    ├───libs
    │   ├───arm64-v8a
    │   │       gdb.setup
    │   │       gdbserver
    │   │       hello_world
    │   ├───armeabi-v7a
    │   │       gdb.setup
    │   │       gdbserver
    │   │       hello_world
    │   ├───x86
    │   │       gdb.setup
    │   │       gdbserver
    │   │       hello_world
    │   └───x86_64
    │           gdb.setup
    │           gdbserver
    │           hello_world
    │           
    └───obj
        └───local
            ├───arm64-v8a
            │   │   hello_world
            │   ├───objs
            │   │   └───hello_world
            │   └───objs-debug
            │       └───hello_world
            │               main.o
            │               main.o.d
            ├───armeabi-v7a
            │   │   hello_world
            │   │   
            │   ├───objs
            │   │   └───hello_world
            │   └───objs-debug
            │       └───hello_world
            │               main.o
            │               main.o.d
            ├───x86
            │   │   hello_world
            │   │   
            │   ├───objs
            │   │   └───hello_world
            │   └───objs-debug
            │       └───hello_world
            │               main.o
            │               main.o.d
            └───x86_64
                │   hello_world
                │   
                ├───objs
                │   └───hello_world
                └───objs-debug
                    └───hello_world
                            main.o
                            main.o.d


### Вид настроек проекта из GUI C::B:

----------

![Image1](img/Image1.png)

![Image2](img/Image2.png)

![Image3](img/Image3.png)

![Image4](img/Image4.png)

Обязательно указать возможные пути где расположены объектные файлы с отладочными символами:

- `obj/local/armeabi-v7a`  
- `obj/local//arm64-v8a`  
- `obj/local/x86`  
- `obj/local/x86_64`  

![Image5](img/Image5.png)

Необходимо добавить команды `GDB` передающие информацию о нахождении отладочных символов для платформы подключенного устройства:  

- `set solib-search-path obj/local/armeabi-v7a` - расположение отладочных символов для активного устройста.  
- `file obj/local/armeabi-v7a/<имя приложения>`  - имя отлаживаемого приложения.  


### Вид настроек компилятора в C::B:

----------

![Image6](img/Image6.png)

![Image7](img/Image7.png)

![Image8](img/Image8.png)


### Дополнительное меню отладки NDK приложения:

----------

Оба используемых в меню скрипта имеют фиксированное имя и генерируются автоматически, во время исполнения `Makefile`, для удобства разумно добавить их в меню:

![Image11](img/Image11.png)

![Image12](img/Image12.png)

Метод отладки приложения состоит из типовых действий, например через `F8` или меню `Отладка -> Старт`. Как только запустился дебагер, вам надо вызвать из созданного меню пункт `ADB Debug Remote server`, этой командой вы запустите `GDB сервер` на устройстве, который запустит ваше приложение. Вы подключаетесь к `GDB серверу` дистанционно и можете проводить сеанс отладки.  
В режиме `Debug` запуск дистанционного сервера происходит автоматически и не требует вызова данного пункта меню.  
Смотрите скриншот настроек дебагера в проекте выше.

Данная схема запуска отладки поддерживает и не рутированные устройства.

### Глобальные настройки дебагера для NDK выглядят так:

----------

![Image13](img/Image13.png)


### Отладка на устройстве:

----------

![Image14](img/Image14.png)


### Результат сборки приложения толчейном NDK:

----------

![Image10](img/Image10.png)

Преимущества использования NDK в отличии от статической сборки под определенную платформу:

- Не нужно компилировать двоичные файлы статически, поэтому размер выходного двоичного файла будет меньше.  
- Можно использовать android `C/C++` библиотеки, такие как liblog, чтобы иметь возможность иметь вывод в logcat из аппликации.  

### Скачать:

----------

Полный код `NDK C::B template` находиться в директории [C::B шаблон](https://github.com/ClnViewer/Code-Blocks-Android-NDK/tree/master/CB-Template/)  
Страница [инсталлятора](CBNDKAUTOINSTALL.RU.md) `NDK C::B template`  

- Скачать инсталлятор `NDK C::B template` [CodeBlocksNdkTemplate v.0.0.3.15 /win32 /win64 (13.02.2020)](https://clclon.github.io/Code-Blocks-Android-NDK/CodeBlocksNdkTemplate.exe)  
- Скачать утилиту конвертации настроек [cbp2ndk v.0.0.3.15 /win32 /win64 (13.02.2020)](https://clclon.github.io/Code-Blocks-Android-NDK/cbp2ndk.zip)  
- Скачать утилиту очистки бинарного файла `armeabi-v7a` [android-elf-cleaner v.0.0.3.15 /win32 /win64 (13.02.2020)](https://clclon.github.io/Code-Blocks-Android-NDK/android-elf-cleaner.zip)  


Обсуждение на [ru.stackoverflow.com](https://ru.stackoverflow.com/questions/972826/codeblocks-android-ndk/972896#972896)


## License

_MIT_
