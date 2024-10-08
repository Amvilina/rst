Глава 4. Соединение сервисов
----------------------------

  Обмен сообщениями между сервисами осуществляется с помощью стандартного ``IP``, таким образом, сервисы становятся независимыми друг от друга и могут крутиться на разных узлах. Самыми распространёнными на сегодняшний момент являются два протокола обмена данными между процессами: ``TCP`` и ``UDP``, они же реализованы и в библиотеке ``tll``

|

UDP - User Datagram Protocol
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

	- Является более простой версией по сравнению с ``TCP``, поэтому начнём с него
	- Сам протокол состоит из заголовка ( порт отправителя, порт получателя, длина данного сообщения, контрольная сумма ) и передаваемых данных. Каждое поле заголовка занимает 2 байта, весь заголовок - 8
	- Протокол **НЕ ПОДРАЗУМЕВАЕТ** 

	  - Проверку существования получателя сообщения с конкретным адрес-портом
	  - Проверку готовности получателя принимать сообщения
	  - Проверку корректности принятых данных получателем
	  - Проверку того, что получатель принимает данные в таком же порядке, в котором они были отправлены
	  - Проверку того, что отправленные пакеты не потерялись
	  
	- Грубо говоря, ``UDP`` позволяет отправлять по конкретному адрес-порту сообщения в надежде на то, что их там корректно примут, с осознанием того, что часть сообщений может быть потеряна или доставлена в другом порядке
	- Про протокол можно почитать `по ссылке <https://datatracker.ietf.org/doc/html/rfc768>`_

TCP - Transmission Control Protocol
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

	- Более сложный протокол, который **ПОДРАЗУМЕВАЕТ** и **ПОЗВОЛЯЕТ** выполнение функций, описанных в ``UDP``
	- Протокол состоит из заголовка ( порт отправителя, порт получателя, ... ) и передаваемых данных. Сам заголовок содержит гораздо больше полей, чем ``UDP``, что и позволяет протоколу выполнять более сложные операции. Длина заголовка составляет от 20 до 60 байт
	- Перед тем, как отправлять полезные сообщения получателю, отправитель должен установить с ним соединение с помощью процесса ``рукопожатия``, после которого обе стороны готовы обмениваться данными
	- Получатель подтверждает каждое полученное сообщение, если этого не произошло ( пакет потерян ), то отправить через какое-то время повторит отправку
	- Сообщения содержат порядковый номер, таким образом получатель может восстановить правильный порядок полученных сообщений
	- Окончание сеанса обмена сообщениями тоже происходит с помощью специального процесса, похожего на ``рукопожатие``. Таким образом обе стороны понимают, что обмен данными закончился корректно, а не по причине случайный разрыва соединения
	- Про протокол можно почитать `по ссылке <https://datatracker.ietf.org/doc/html/rfc793>`_

TCP vs UDP
^^^^^^^^^^

  - ``UDP`` гораздо быстрее TCP из-за меньшего размера заголовка и отсутствия процесса подтверждения принятия пакетов
  - ``TCP`` обладает меньшей скоростью, однако огромной отказоустойчивостью. Может гарантировать, что получатель примет именно то, что было отправлено
  - Из-за этих особенностей ``UDP`` часто используется при передаче видео или голоса, потому что потеря отдельных пакетов не отразится на качестве всей картинки / аудиодорожки
  - ``TCP`` же используется везде, где данные очень критичны, в том числе при обмене сообщениями между сервисами. Нам не простят, если сделки клиентов пропадут или придут не в том порядке

Связываем сервисы комиссии и генерации
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Будем использовать канал pub+tcp, который реализует за нас протокол и отношение клиент-сервер

  - Нужно сделать обновления в ``generator.cc`` для обработки подключения клиента:

    .. code:: c++

      // ... ================== ...

      // для обработки сообщений подключение / отключение
      #include <tll/channel/tcp-scheme.h>

      // ... ================== ...

          // теперь к выходному каналу подключается клиент
          // мы можем отслеживать это для какой-то более сложной логики
          int callback_tag(tll::channel::TaggedChannel<Output> * c, const tll_msg_t *msg) {
              
              // если сообщение подключения, то выводим на экран
              if (msg->msgid == tcp_scheme::Connect::meta_id()) 
                  _log.info("client connected!");
              

              // если сообщение отключения, то выводим на экран
              if (msg->msgid == tcp_scheme::Disconnect::meta_id()) 
                  _log.info("client disconnected!");
              
              
              return 0;
          }

      // ... ================== ...




  
  - Теперь обновим конфиги ``generator-processor.yaml``:

    .. code:: yaml

          # ...

            output-channel:
              init:
                tll.proto: pub+tcp      # мы отправляем данные через pub+tcp
                tll.host: ./pub.socket  # можно написать localhost:8080 или любой доступный
                                        # можно просто воспользоваться сокетами в линуксе, т.к. одна машинка
                mode: server            # генератор - сервер, он отправляет данные
                scheme: yaml://./messages/transaction.yaml
                dump: scheme

          # ...


  - Аналогично обновим конфиги ``commission-processor.yaml``:

    .. code:: yaml

          # ...

            input-channel:           
              init:                       
                tll.proto: pub+tcp                 
                tll.host: ./pub.socket  # подключаемся к тому же адресу / сокету
                mode: client            # сервис - клиент, он получает данные
                scheme: yaml://./messages/transaction.yaml 
                dump: yes
              depends: logic  

          # ... 

  - Мы используем ``pub+tcp``, потому что в нём уже реализована логика подключения клиентов и отправки им сообщений. Клиентов можно подключить сколько угодно, а сам сервер просто отправляет сообщения, не заботясь о том, подключён ли кто-то к нему. Если использовать чистый ``tcp``, то нужно будет самостоятельно реализовывать логику: запоминать адреса клиентов; добавлять в каждое сообщение поле addr при его отправке; следить за тем, что клиент действительно подключён, чтобы не получать сообщение об ошибке при отправке. ``pub+tcp`` всё это делает за нас!
  - Для проверки открываем 2 окна терминала и запускаем команды:

    ``$ tll-processor generator-processor.yaml`` 

    ``$ tll-pyprocessor commission-processor.yaml``

  - Нужно запускать сначала сервер / генератор, потому что иначе клиент не поймёт куда подключаться и будет ошибка
  - В логах 2 сервисов будут видны сообщения получения / передачи сообщения
  - Проверить работу можно: ``$ tll-read output.dat --seq 0:2``:

    .. code::

          - seq: 0
            name: Commission
            data:
              time: '2024-09-02T18:55:08.641117473Z'
              id: 1
              value: '499.63'
          
          - seq: 1
            name: Commission
            data:
              time: '2024-09-02T18:55:11.643263104Z'
              id: 2
              value: '268.93'
          
          - seq: 2
            name: Commission
            data:
              time: '2024-09-02T18:55:14.641143454Z'
              id: 3
              value: '33.53'



