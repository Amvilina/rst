Глава 9. Control Unit ( CU )
----------------------------

  Каналами можно управлять с помощью специальных контрольных сообщений. rotate+ - Rotate, generator-block - Block, итд. Эти сообщения каналам / сервисам извне отправляет специальная программа - CU



Создание CU с помощью tll-logic-control
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - ``tll-logic-control`` - специальная логика, которая даёт возможность переадресовать полученное ею сообщение нужному каналу
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
            processor: processor-client # канал для связи с процессором
            input: input-logic          # канал, через который мы будем получать управляющие сообщения
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

        # обработчик сообщений должен быть связан с процессором через ipc
        processor-client:
          init:
            tll.proto: ipc
            mode: client
            master: processor/ipc
            scheme: yaml://../src/logic/control.yaml
            dump: yes
          depends: control-logic

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
          s.post(data={'dest': 'output-channel', 'data': {
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
          s.post(data={'dest': 'output-channel', 'data': {
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


Проверка
^^^^^^^^

  - Запустим наш сервер: ``$ tll-processor generator-processor.yaml``
  - Откроем новое окно терминала и запустим скрипт: ``$ python3 send-block-command``. В терминале должно вывестись сообщение: ``block created``
  - В логках генератора будет:

    .. code::

      2024-09-19 13:41:17.024 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 10, name: Connect, seq: 0, size: 19, addr: 0x16
        host: {unix: 0}
        port: 0
      
      2024-09-19 13:41:17.025 INFO tll.channel.input-logic: Recv message: type: Data, msgid: 4176, name: MessageForward, seq: 0, size: 90, addr: 0x16
        dest: "output-channel"
        data:
          type: Control
          name: "Block"
          seq: 0
          addr: 0
          data: "{"type": "commission-sum"}"
      
      2024-09-19 13:41:17.025 INFO tll.channel.processor-client: Post message: type: Data, msgid: 4176, name: MessageForward, seq: 0, size: 118
        Failed to format message MessageForward field data.data: Offset out of bounds: offset 16777281 > data size 84
        00000000:  26 00 00 00  0f 00 00 01  01 00 64 00  00 00 00 00  &.........d.....
        00000010:  00 00 00 00  00 00 00 00  00 00 00 00  00 00 17 00  ................
        00000020:  00 00 41 00  00 01 6f 75  74 70 75 74  2d 63 68 61  ..A...output-cha
        00000030:  6e 6e 65 6c  00 63 6f 6d  6d 69 73 73  69 6f 6e 2d  nnel.commission-
        00000040:  73 75 6d 00  00 00 00 00  00 00 00 00  00 00 00 00  sum.............
        00000050:  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ................
        00000060:  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  ................
        00000070:  00 00 00 00  00 00                                  ......
      
      2024-09-19 13:41:17.025 INFO tll.channel.input-logic: Post message: type: Data, msgid: 40, name: Ok, seq: 0, size: 0, addr: 0x16
        
      2024-09-19 13:41:17.025 INFO tll.channel.output-channel: Post message: type: Control, msgid: 100, name: Block, seq: 0, size: 64
        {type: "commission-sum"}

      # ... block-channel logs ...

      2024-09-19 13:41:17.026 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 20, name: Disconnect, seq: 0, size: 0, addr: 0x16

      # ...
  - Если запустить этот скрипт несколько раз, то у нас появится несколько срезов: ``$ ls blocks-storage/`` -> ``block.1.dat  block.2.dat  block.3.dat  block.4.dat``
  - Аналогично можно проверить и команду ``Rotate``: ``$ python3 send-rotate-command``. В терминале: ``file rotated``
  - В логах генератора:

    .. code::

      2024-09-19 13:41:24.745 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 10, name: Connect, seq: 0, size: 19, addr: 0x100000016
        host: {unix: 0}
        port: 0
      
      2024-09-19 13:41:24.746 INFO tll.channel.input-logic: Recv message: type: Data, msgid: 4176, name: MessageForward, seq: 0, size: 65, addr: 0x100000016
        dest: "output-channel"
        data:
          type: Control
          name: "Rotate"
          seq: 0
          addr: 0
          data: ""
      
      2024-09-19 13:41:24.746 INFO tll.channel.processor-client: Post message: type: Data, msgid: 4176, name: MessageForward, seq: 0, size: 54
        Failed to format message MessageForward field data.name: Offset out of bounds: offset 150 > data size 44
        00000000:  26 00 00 00  0f 00 00 01  01 00 96 00  00 00 00 00  &...............
        00000010:  00 00 00 00  00 00 00 00  00 00 00 00  00 00 17 00  ................
        00000020:  00 00 01 00  00 01 6f 75  74 70 75 74  2d 63 68 61  ......output-cha
        00000030:  6e 6e 65 6c  00 00                                  nnel..
      
      2024-09-19 13:41:24.746 INFO tll.channel.input-logic: Post message: type: Data, msgid: 40, name: Ok, seq: 0, size: 0, addr: 0x100000016
        
      2024-09-19 13:41:24.746 INFO tll.channel.output-channel: Post message: type: Control, msgid: 150, name: Rotate, seq: 0, size: 0

      # ... rotate+file logs ...

      2024-09-19 13:41:24.747 INFO tll.channel.input-logic: Recv message: type: Control, msgid: 20, name: Disconnect, seq: 0, size: 0, addr: 0x100000016

      # ... 

  - Если запустить этот скрипт несколько раз, то у нас появится несколько файлов с данными: ``$ ls storage/`` -> ``output.16.dat  output.20.dat  output.5.dat  output.current.dat``





