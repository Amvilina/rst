Глава 9. Control Unit ( CU )
----------------------------

  Каналами можно управлять с помощью специальных контрольных сообщений. rotate+ - Rotate, generator-block - Block, итд. Эти сообщения каналам / сервисам извне отправляет специальная программа - CU



Создание CU с помощью tll-logic-control
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - ``tll-logic-control`` - специальная логика, которая даёт возможность переадресовать полученное её сообщение
  - Ей отправляется специальное сообщение ``MessageForward``, которое хранит в себе адресата (``dest``) и само сообщение
  - Опишем логику в процессоре нашего генератора, ``generator-processor.yaml``:

    .. code:: yaml

      # ...

      processor.module:
        # ...
        - module: tll-logic-control # добавляем модуль
                                    # логика - не часть tll, поэтому её нужно отдельно подключать

      processor.objects:

        # логика, которая отвечает за обработку управляющих сообщений
        control-logic:
          init:
            tll.proto: control
          channels:
            processor: generator # наш основной процесс
            input: input-logic   # канал, через который мы будем получать управляющие сообщения
          depends: generator
        
        # получать мы будем сообщения через простой tcp-канал
        # сервер будет слушать подключения через сокет и ждать команд
        input-logic:
          init:
            tll.proto: tcp
            tll.host: ./pub-logic.socket
            scheme: yaml://../src/logic/control.yaml
            mode: server
            dump: yes

        # ...

Скрипт отправки управляющих сообщений
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Напишем скрипт на питоне, который при запуске подключается через tcp к серверу (``pub-logic.socket``) и отправляет ему специальное сообщение
  - ``send-block-command.py``:

    .. code:: python

      import tll 
      import tll.channel as C
      from tll import asynctll # для запуска асинхронных методов
      import tll.logger        # для настраивания режимов логирования
      
      # уберём логи из нашей программы, чтобы они не мешали при каждом запуске
      # устанавливая уровень 'WARNING', мы будем получать сообщение только о предупреждениях и ошибках
      tll.logger.init()
      tll.logger.configure({'levels': {'tll.channel.asynctll':'WARNING'}})
      tll.logger.configure({'levels': {'tll.context':'WARNING'}})

      # получаем контекст
      ctx = C.Context()
      
      # создаём асинхронный луп
      loop = asynctll.Loop(ctx)
      
      # объявляем функцию для создания канала и отправки сообщения
      async def main(loop):

          # создаём канал: tcp-клиент
          s = loop.Channel("tcp://./pub-logic.socket;mode=client;name=client;scheme=yaml://../src/logic/control.yaml", context = ctx)

          # пытаемся открыть канал
          try:
              s.open()
          except:

              # сервер может быть не запущен, поэтому выходим из функции
              print("can't connect")
              return

          # отправляем сообщение MessageForward, в 'dest' указываем нужного получателя
          # в 'data' просто записано сообщение, которое мы хотим передать
          # сообщение 'Block' с полем 'type': 'commission-sum'
          s.post(data={'dest': 'generator', 'data': {
                      'type': 'Control',
                      'name': 'Block',
                      'seq': 0,
                      'addr': 0,
                      'data': b'{"type": "commission-sum"}'
                  }}, 
                  name='MessageForward')
      
          # ждём ответа от сервера
          answer = await s.recv()

          # сервер, если получилось переслать сообщение, вернёт сообщение 'Ok'
          if s.unpack(answer).SCHEME.name == 'Ok':
              print("block created")
          else:
              print("not created!!!")

          # закрываем канал
          s.close()
      
      # запускаем наш асинхронный луп
      loop.run(main(loop))

  - Создадим аналогичный скрипт для того, чтобы управлять хранилищем ( специальное сообщение Rotate ), ``send-rotate-command.py``:

    .. code:: python

      import tll 
      import tll.channel as C
      from tll import asynctll
      import tll.logger
      
      tll.logger.init()
      tll.logger.configure({'levels': {'tll.channel.asynctll':'WARNING'}})
      tll.logger.configure({'levels': {'tll.context':'WARNING'}})
      ctx = C.Context()
      
      loop = asynctll.Loop(ctx)
      
      async def main(loop):
          s = loop.Channel("tcp://./pub-logic.socket;mode=client;name=client;scheme=yaml://../src/logic/control.yaml", context = ctx)
          try:
              s.open()
          except:
              print("can't connect")
              return
          s.post(data={'dest': 'generator', 'data': {
                      'type': 'Control',
                      'name': 'Rotate',
                      'seq': 0,
                      'addr': 0,
                      'data': ''
                  }}, 
                  name='MessageForward')
      
          answer = await s.recv()
          if s.unpack(answer).SCHEME.name == 'Ok':
              print("file rotated")
          else:
              print("not rotated!!!")
          s.close()
      
      loop.run(main(loop))

Обработка входных сообщений в генераторе
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  
  - ``control-logic`` будет отправлять приходящие ему сообщения ( из скриптов ) в генератор, их нужно корректно обработать
  - ``generator.cc``:

    .. code:: c++

      // ...

          int _init(const tll::Channel::Url &, tll::Channel *master) {

              // ...

              // схема нашего сервиса будет совпадать со схемой _output ( stream-server )
              // этот канал уже знает про приходящие контрольные сообщения (Block и Rotate)
              _scheme_control.reset(_output->scheme(TLL_MESSAGE_CONTROL));
          }

          int _post(const tll_msg_t *msg, int flags) {

              // проверяем, что пришли данные
              if (msg->type != TLL_MESSAGE_DATA) 
                  return 0;
                  
              // проверяем, что сообщение именно 'MessageForward'
              if (msg->msgid != processor_scheme::MessageForward::meta_id())
                  return 0;
      
              // конвертируем сообщение в нужную структуру
              auto data = processor_scheme::MessageForward::bind(*msg);

              // получаем отправленные данные
              auto message = data.get_data();

              // создаём новое сообщение, копирую данные
              tll_msg_t m = {};
              m.type = message.get_type();
              m.msgid = message.get_msgid();
              m.seq = message.get_seq();
              m.addr.u64 = message.get_addr();
              m.data = message.get_data().data();
              m.size = message.get_data().size();

              // отправляем его на выход ( stream-server )
              // stream-server затем отправляет все контрольные сообщения своим каналам
              _output->post(&m);

              return 0;
          }

      // ..


Проверка
^^^^^^^^

  - Запустим наш сервер: ``$ tll-processor generator-processor.yaml``
  - Откроем новое окно терминала и запустим скрипт: ``$ python3 send-block-command``. В терминале должно вывестись сообщение: ``block created``
  - В логках генератора будет:

    .. code::

      2024-09-18 21:22:25.227 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 10, name: Connect, seq: 0, size: 19, addr: 0x13
        host: {unix: 0}
        port: 0

      2024-09-18 21:22:25.228 INFO tll.channel.input-logic: Recv message: type: Data, msgid: 4176, name: MessageForward, seq: 0, size: 85, addr: 0x13
        dest: "generator"
        data:
          type: Control
          name: "Block"
          seq: 0
          addr: 0
          data: "{"type": "commission-sum"}"

      2024-09-18 21:22:25.228 INFO tll.channel.output-channel: Post message: type: Control, msgid: 100, name: Block, seq: 0, size: 64
        {type: "commission-sum"}

      # ... block-channel logs ...

      2024-09-18 21:22:25.229 INFO tll.channel.input-logic: Post message: type: Data, msgid: 40, name: Ok, seq: 0, size: 0, addr: 0x13
  
      2024-09-18 21:22:25.229 INFO tll.channel.input-logic/19: State change: Active -> Closing
      2024-09-18 21:22:25.229 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 20, name: Disconnect, seq: 0, size: 0, addr: 0x13

      # ...
  - Если запустить этот скрипт несколько раз, то у нас появится несколько срезов: ``$ ls blocks-storage/`` -> ``block.1.dat  block.2.dat  block.3.dat  block.4.dat``
  - Аналогично можно проверить и команду ``Rotate``: ``$ python3 send-rotate-command``. В терминале: ``file rotated``
  - В логах генератора:

    .. code::

      2024-09-18 21:28:52.313 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 10, name: Connect, seq: 0, size: 19, addr: 0x400000013
        host: {unix: 0}
        port: 0

      2024-09-18 21:28:52.314 INFO tll.channel.input-logic: Recv message: type: Data, msgid: 4176, name: MessageForward, seq: 0, size: 60, addr: 0x400000013
        dest: "generator"
        data:
          type: Control
          name: "Rotate"
          seq: 0
          addr: 0
          data: ""
      
      2024-09-18 21:28:52.314 INFO tll.channel.output-channel: Post message: type: Control, msgid: 150, name: Rotate, seq: 0, size: 0

      # ... rotate+file logs ...

      2024-09-18 21:28:52.314 INFO tll.channel.input-logic: Post message: type: Data, msgid: 40, name: Ok, seq: 0, size: 0, addr: 0x400000013
  
      2024-09-18 21:28:52.315 INFO tll.channel.input-logic/19: State change: Active -> Closing
      2024-09-18 21:28:52.315 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 20, name: Disconnect, seq: 0, size: 0, addr: 0x400000013

      # ... 

  - Если запустить этот скрипт несколько раз, то у нас появится несколько файлов с данными: ``$ ls storage/`` -> ``output.16.dat  output.20.dat  output.5.dat  output.current.dat``





