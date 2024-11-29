Глава 10. Resolve
-----------------

  Resolve - удобный способ подключать клиентов ( потребителей данных ) к серверу ( производителю данных ), не зная заранее через какой канал будет происходить связь

Resolve overview
^^^^^^^^^^^^^^^^

  - Для удобства создаётся еще один сервис - ``Resolve``, к которому могут подключаться потребители и производители данных
  - Производитель при подключении передаёт своё имя и имена каналов, доступных для подключения. Он формирует конфиг для создания каждого канала на стороне потребителя и тоже отправляет его в ``Resolve``
  - Если у производителя есть канал ``output: tcp (mode: server, host: abc)``, то в конфиге он передаст ``tcp (mode: client, host: abc)``, чтобы потребитель смог корректно подключиться
  - Потребитель при подключении к ``Resolve`` запрашивает сервис и его канал для подключения. ``Resolve`` ищет его и отправляет конфиг для создания канала
  - Это очень удобная и гибкая система, если когда-то производитель перейдёт на другой канал для отправки данных, то все клиенты автоматически будут подключаться правильно, потому что ``Resolve`` всё сделает за них

Resolve сервер
^^^^^^^^^^^^^^

  - Сервер представляет питоновскую логику, ``resolve-processor.yaml``:

    .. code:: yaml

      processor.objects:

        # объект - resolve, который описан в классе Resolve ( написан уже за нас )
        resolve:
          url:
            tll.proto: python
            python: tll.channel.resolve:Resolve

          # есть один канал, через который происходит взаимодействие с сервером
          channels: {input: input}
      
        # канал представляет из себя простой tcp-сервер, который ждёт запросов по адресу resolve.sock
        input.url:
          tll.proto: tcp
          tll.host: ./resolve.sock
          af: unix # как расшифровывать tll.host
                   # по умолчанию стоит 'any' - автоматическое
                   # 'unix' - unix-socket, 'ipv4' - ip4, 'ipv6' - ip6
                   # если в host встречается знак '/', то автоматически выбирается 'unix'
          mode: server
          scheme: yaml://../src/logic/resolve.yaml
          dump: yes

Resolve производитель
^^^^^^^^^^^^^^^^^^^^^

  - Поменяем файл процессора нашего генератора, чтобы он работал с ``Resolve``, ``generator-processor.yaml``:

    .. code:: yaml

      # ...

      processor.objects:
       control:
          init:
            tll.proto: control
            service: generator            # сообщаем название нашего сервиса
          channels:
            processor: control-processor
            resolve: control-resolve      # добавляем ещё один канал для связи с resolve
            input: control-input
          dump: yes

          # control будет отправлять данные в processor и resolve, поэтому зависит от них
          depends: control-processor, control-resolve
        
        control-resolve:

          # простой tcp-клиент, который подключается к resolve-серверу
          init:
            tll.proto: tcp
            tll.host: resolve.sock
            mode: client
            af: unix
            scheme: yaml://../src/logic/resolve.yaml
          dump: yes

        # ...

        output-channel:
          init:

            # ...

            # добавляем новый раздел
            tll.resolve:
              export: yes                 # сообщаем, что этот канал нужно экспортировать в Resolve
              export-name: output-channel # имя этого канала
          depends: control

Resolve потребитель
^^^^^^^^^^^^^^^^^^^

  - Поменяем файл процессора нашего сервиса с комиссиями6 чтобы он работал с ``Resolve``, ``commission-processor.yaml``:

    .. code:: yaml

      # ...

      # в этом разделе мы описываем способ подключения к resolve
      processor.defaults:
        resolve.request:

          # это тоже простой tcp-клиент, который подключается к resolve.sock
          tll.proto: tcp
          tll.host: ./resolve.sock
          mode: client
          af: unix
          scheme: yaml://../src/logic/resolve.yaml
          dump: yes

      processor.objects:
        input-channel:
          init:

            # раньше тут было stream+pub+tcp, и описание способа подключения
            # сейчас это заменяется специальным каналом resolve, который автоматически всё будет делать
            tll.proto: resolve
            resolve:
              service: generator      # название сервиса
              channel: output-channel # название канала у сервиса

            dump: yes
          open: !link /sys/logic/info/input-open-params
          depends: logic

      # ...

Проверка
^^^^^^^^

  - Запустим для начала наш ``Resolve``: ``$ tll-pyprocessor resolve-processor.yaml``
  - В другом окне терминала запустим производителя данных: ``$ tll-processor generator-processor.yaml``
  - В логах ``Resolve`` увидим:

    .. code::

      INFO : tll.channel.input: Recv message: type: Control, msgid: 10, name: Connect, seq: 0, size: 19, addr: 0x10000000f
        host: {unix: 0}
        port: 0
      
      INFO : tll.channel.input: Recv message: type: Data, msgid: 10, name: ExportService, seq: 0, size: 61, addr: 0x10000000f
        service: "generator"
        tags: []
        host: "debian-gnu-linux-12.(none)"
      
      INFO : tll.channel.resolve: Register service 'generator', tags []
      INFO : tll.channel.input: Recv message: type: Data, msgid: 40, name: ExportChannel, seq: 0, size: 831, addr: 0x10000000f
      service: "generator"
      channel: "output-channel"
      tags: []
      host: ""
      config:
        - key: "init.af"
          value: "unix"
        - key: "init.mode"
          value: "client"
        - key: "init.request.af"
          value: "unix"
        - key: "init.request.mode"
          value: "client"
        - key: "init.request.tll.host"
          value: "./request.socket"
        - key: "init.request.tll.proto"
          value: "tcp"
        - key: "init.scheme"
          value: "sha256://0d068da18586a4c5415cfa76ea8666beb0be309707e78e1f1dc7fabea0a98ff1"
        - key: "init.tll.host"
          value: "./pub.socket"
        - key: "init.tll.proto"
          value: "stream+pub+tcp"
        - key: "scheme.sha256://0d068da18586a4c5415cfa76ea8666beb0be309707e78e1f1dc7fabea0a98ff1"
          value: "yamls+gz://eJzNj70KwkAQhHufYrttDJggKa71FezlyF1gIfdD9k4MIe/uHiSiKQQ7ux12Zj6mAq+dVYCX4BwxU/B4ACCjoDnJ0ZMdDCu5ACqYV3MiZ/EIaYpFkU/tWWSISeKsYMbRchhykSgGz/LF4sY1fItBUrgsu2Iy+9q9466H/J29cXp6WNMURrWNvI7as+7S28r6L1fGkbpfV342dCEL+tWQpaJuhfME4+OWRQ=="

      INFO : tll.channel.resolve: Register channel generator/output-channel

  - В ещё одном окне терминала запустим потребителя данных: ``$ tll-pyprocessor commission-processor.yaml``
  - Данные начнут поступать, как и раньше, это будет видно в логах. В логах же ``Resolve`` наблюдаем следующее:

    .. code::

      INFO : tll.channel.input: Recv message: type: Control, msgid: 10, name: Connect, seq: 0, size: 19, addr: 0x200000010
        host: {unix: 0}
        port: 0
      
      INFO : tll.channel.input: Recv message: type: Data, msgid: 60, name: Request, seq: 0, size: 41, addr: 0x200000010
        service: "generator"
        channel: "output-channel"
      
      DEBUG: tll.channel.resolve: Request for generator/output-channel
      INFO : tll.channel.resolve: Reply with channel generator/output-channel to input:200000010
      INFO : tll.channel.input: Post message: type: Data, msgid: 40, name: ExportChannel, seq: 0, size: 858, addr: 0x200000010
        service: "generator"
        channel: "output-channel"
        tags: []
        host: "debian-gnu-linux-12.(none)"
        config:
          - key: "init.af"
            value: "unix"
          - key: "init.mode"
            value: "client"
          - key: "init.request.af"
            value: "unix"
          - key: "init.request.mode"
            value: "client"
          - key: "init.request.tll.host"
            value: "./request.socket"
          - key: "init.request.tll.proto"
            value: "tcp"
          - key: "init.scheme"
            value: "sha256://0d068da18586a4c5415cfa76ea8666beb0be309707e78e1f1dc7fabea0a98ff1"
          - key: "init.tll.host"
            value: "./pub.socket"
          - key: "init.tll.proto"
            value: "stream+pub+tcp"
          - key: "scheme.sha256://0d068da18586a4c5415cfa76ea8666beb0be309707e78e1f1dc7fabea0a98ff1"
            value: "yamls+gz://eJzNj70KwkAQhHufYrttDJggKa71FezlyF1gIfdD9k4MIe/uHiSiKQQ7ux12Zj6mAq+dVYCX4BwxU/B4ACCjoDnJ0ZMdDCu5ACqYV3MiZ/EIaYpFkU/tWWSISeKsYMbRchhykSgGz/LF4sY1fItBUrgsu2Iy+9q9466H/J29cXp6WNMURrWNvI7as+7S28r6L1fGkbpfV342dCEL+tWQpaJuhfME4+OWRQ=="

      DEBUG: tll.channel.input/16: Connection closed
      INFO : tll.channel.input/16: State change: Active -> Closing
      INFO : tll.channel.input: Recv message: type: Control, msgid: 20, name: Disconnect, seq: 0, size: 0, addr: 0x200000010
  

