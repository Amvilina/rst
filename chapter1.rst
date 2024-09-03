Глава 1. Настройка окружения
----------------------------
  

 Использую Debian Linux 12 ARM64


|



Устанавливаем гит
^^^^^^^^^^^^^^^^^

  - ``$ sudo apt install git``
  - После команды система запросит пароль пользователя, при вводе пароль не будет отображаться
  - После ввода нажать ``Enter``



Клонируем нужные репозитории
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - ``$ git clone https://github.com/shramov/tll.git``
  - ``$ git clone https://github.com/shramov/tll-lua.git``
  - После этих команд у нас появляются директории: ``tll`` и ``tll-lua``



Устанавливаем нужные библиотеки
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Устанавливаем инструменты разработчика: ``$ sudo apt install build-essential devscripts dpkg python3``
  - Терминал попросит нас ввести: ``Do you want to continue? [Y/n]``, вводим ``y``
  - Переходим в директорию библиотеки: ``$ cd tll``
  - Устанавливаем все необходимые библиотеки: ``/tll$ sudo apt satisfy "`python3 -c 'from debian.deb822 import Deb822; print(Deb822(open("debian/control"))["Build-Depends"])'`"``
  - Переходим в директорию ``tll-lua``: ``/tll$ cd ../tll-lua`` и запускаем аналогичную команду
 


Собираем tll фреймворки
^^^^^^^^^^^^^^^^^^^^^^^

  - После установки библиотек запускаем в 2-х директориях команды: 

    - ``/tll$ DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -uc -b -j4``
    - ``/tll-lua$ DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -uc -b -j4``
  - После успешных выполнений этих команд у нас должны появиться файлы ``*.deb`` в корневой директории``
  - Устанавливаем все пакеты: ``$ sudo dpkg -i *.deb``
  - Проверяем всё командами: 

    - ``$ tll-processor`` - вывод: ``ERROR: tll.config.yaml: Failed to open file '': No such  ...``
    - ``$ tll-read`` - вывод: ``usage: tll-read [-h] ...``
    - ``$ python3``, ``>>> import tll`` - вывод: отсутсвие ошибок, т.е. ожидание следующей команды ``>>>``