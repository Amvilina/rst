Глава 6. Знакомство с каналом stream+
-------------------------------------

  Парктически все сервисы в продакшене связаны между собой с помощью ``stream+`` каналов ( например, ``stream+pub+tcp`` ). Ниже будут подробнее описаны возможности данного канала и причины его использования. Для простоты будем рассматривать случай именно ``stream+pub+tcp``

|

Overview stream+
^^^^^^^^^^^^^^^^

  - ``stream+`` - оболочка вокруг стандартных каналов связи, которая позволяет расширить возможности обмена данными и автоматизировать этот процесс
  - Канал на самом деле состоит из 2-х: ``online`` и ``request``
  - Участников стоит разделить по принципу работы на ``mode: server / client``
  - ``online`` ( ``stream+pub+tcp`` ) - постоянный канал связи с отношением 1 сервер : много клиентов
  - ``request`` ( ``stream+tcp`` ) - временный канал связи, который каждый клиент открывает с сервером для получения специфичных данных. Этот канал открывается только в моменты открытия клиента ( вызове функции ``_open(...)`` )

Что может stream+ сервер?
^^^^^^^^^^^^^^^^^^^^^^^^^

  - Через канал ``online`` сервер транслирует нужную информацию всем клиентам, которые подключены к нему. Принцип работы 1 в 1, как с простым ``pub+tcp``
  - Одновременно с трансляцией сервер сохраняет все отправленные сообщение в специальное хранилище, которое чаще всего организовано с помощью ``rotate+file`` ( будет описано ниже )
  - При подключении клиента к каналу ``request`` ( для каждого клиента свой ), сервер отправляет специально запрошенные данные от клиента ( будут описаны в главе для клиента ), при этом продолжая транслировать свои сообщения в ``online``

Что может stream+ клиент?
^^^^^^^^^^^^^^^^^^^^^^^^^

  - Через канал ``online`` клиент подключается к серверу, после чего получает возможность 'в реальном времени' получать данные от сервера
  - В момент открытия ( вызове функции ``_open(...)`` ) клиент может открыть канал ``request`` с сервером для того, чтобы запросить специальные данные. Сам канал скрыт от логики, передача всех специальных данных ( срезов или исторических данных ) происходит 'под катом'
  - Если клиенту не нужно никакой специальной информации от сервера, т.е. ему требуется просто подключиться к каналу ``online``, то для этого в параметрах открытия стоит прописать: ``open: {mode: online}``. В таком случае клиент превратится в обычного клиента ``pub+tcp``
  - Клиент имеет возможность запросить старые данные у сервера, начиная с какого-то ``seq``. Для этого в параметрах открытия стоит прописать: ``open: {mode: seq, seq: 100}``. При открытии канала в этом режиме происходит следующая последовательность действий:

    - Клиент подключается к каналу ``online``, чтобы иметь возможность получать сообщения
    - Клиент открывает канал ``request`` и отправляет запрос серверу, чтобы получить все сообщения, начиная с запрошенного ``seq``
    - Сервер в ответ на этот запрос отправляет специальное сообщение ``Reply``, где записан ``last_seq`` последнего сообщения, которое было отправлено через канал ``online``
    - Клиент начинает получать сообщения из ``online`` и записывать их в специальный буфер
    - Сервер отправлет клиенту сообщения через канал ``request``. Если ``seq=100`` и ``last_seq=130``, то сервер отправит сообщения с номерами: ``seq in [100, 130]``
    - Клиент получает и обрабатывает все приходящие исторические сообщения ( сначала от сервера через ``request``, затем накопленные из буфера )
    - После окончания обработки всех сообщений клиент генерирует специальное сообщение ``Online``, закрывает канал ``request`` и переходит к режиму обработки сообщений, получаемых из канала ``online`` ( т.е. превращается в простого клиента ``pub+tcp`` )
  - Ещё можно открыть канал ``request`` с параметром ``seq-data``, который инкрементирует первый ``seq``. Следующие 2 записи 'аналогичны': ``open: {mode: seq, seq: 101}`` и ``open: {mode: seq-data, seq: 100}`` 
  - Обработка исторических и онлайн сообщений ( до и после получения специального сообщения ``Online`` ) часто очень отличается. Пример: заявки клиентов, полученные в 'историческом' режиме будут отклоняться, а в 'онлайн' - обрабатываться

Про rotate+file
^^^^^^^^^^^^^^^

  - ``rotate+`` используется для построения системы хранения большого числа информации ( логи / сообщения ) 
  - Хранить все данные в одном файле - не самое эффективное решение. Именно поэтому ``rotate+`` разбивает один большой файл с информацией на много небольших, данные в которых хранятся в зависимости от ``seq`` хранимого сообщения
  - ``rotate+`` внутри себя имеет ``map``, с помощью которой на любое запрошенное сообщение с конкретным ``seq`` выберет нужный файл, а затем в нём найдёт необходимое сообщение
  - Предположим, что ``tll.host: storage/output``. Все файлы имеют определённые названия: ``storage/output.0.dat``, ``storage/output.100.dat``, ... , ``storage/output.current.dat`` - в данном примере каждый файл состоит из 100 сообщений, на практике используется гораздо больший размер, который может не быть одинаковым
  - ``0.dat`` означает, что в нём хранятся сообщения с индексами ``0, 1, 2, ... 99``; 100.dat - ``100, 101, ...``
  - В файле ``current.dat`` хранятся самые последние сообщения, число которых ещё не достигло нужного, чтобы их сформировать в отдельный файл
  - Такая архитектура позволяет быстрее находить нужные нам сообщения, потому что данные хранятся древовидным образом
  - ``rotate+`` является просто инструментом, с помощью которого можно добиться желаемого результута. При получении специального сообщения ``Rotate`` происходит создание нового файла. Когда конкретно отправлять специальное сообщение и, соответственно, разбивать данные на файлы решает логика 'выше' ``rotate+``

Тестирование stream+
^^^^^^^^^^^^^^^^^^^^

  - Свяжем наши 2 сервиса: комиссионный и генератора с помощью ``stream+pub+tcp``
  - ``generator-processor.yaml``:

    .. code:: yaml

      # ...

        output-channel:

          # generator - сервер / производитель данных
          init:
            tll.proto: stream+pub+tcp # здесь добавляется просто stream+
            tll.host: ./pub.socket
            mode: server
            autoseq: true
            dump: yes
            scheme: yaml://./messages/transaction.yaml

            # новые поля, обязательные для stream+:

            # через этот канал будет осуществляться связь с отдельными клиентами
            # мы используем просто tcp, потому что каждый клиент захочет получать свои данные
            # нам здесь не нужна трансляция данных, как в pub+tcp
            request: tcp://./request.socket
            
            # хранилище, где будут храниться все отправленные сообщения
            # обязательно должна быть директория, поэтому перед запуском её нужно создать:
            # $ mkdir storage
            storage: rotate+file://storage/output
          
      # ...

  - ``commission-processor.yaml``:

    .. code:: yaml

      # ...

        input-channel:

          # commission - клиент / потребитель данных
          init:
            tll.proto: stream+pub+tcp
            tll.host: ./pub.socket 
            mode: client         
            request: tcp://./request.socket
            scheme: yaml://./messages/transaction.yaml
            dump: yes

          # параметры открытия канала
          open:
            mode: online  # проверим сначала в online режиме

          depends: logic
          
      # ...

  - Запустим сначала сервер: ``$ tll-processor generator-processor``, подождём пока он сгенерирует данные. Затем запустим клиента в другом окне терминала: ``$ tll-pyprocessor commission-processor``. В итоге мы начнём получать онлайн данные ( в нашем случае, начиная с ``seq = 10`` ), не имея доступа к старым:

    .. code::

      2024-09-09 17:06:21.039 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 10, name: Transaction, seq: 10, size: 26
        time: 2024-09-09T14:06:21.038702405
        id: 11
        price: 97.21
        count: 45
      
      2024-09-09 17:06:21.040 INFO tll.channel.output-channel: Post message: type: Data, msgid: 20, name: Commission, seq: 10, size: 24
        time: 2024-09-09T14:06:21.038702405
        id: 11
        value: 43.74

      ...

  - Можем не останавливать наш сервер. Изменим параметры открытия канала у клиента:

    .. code:: yaml

      # ...

          open:
            mode: seq
            seq: 0     # получим все данные, начиная с самого начала

      # ...

  - И снова запустим клиента: ``$ tll-pyprocessor commission-processor``:

    .. code::

      2024-09-09 17:09:45.479 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 10, name: Transaction, seq: 0, size: 26
        time: 2024-09-09T14:06:11.037913425
        id: 1
        price: 358.02
        count: 42
      
      2024-09-09 17:09:45.480 INFO tll.channel.output-channel: Post message: type: Data, msgid: 20, name: Commission, seq: 0, size: 24
        time: 2024-09-09T14:06:11.037913425
        id: 1
        value: 150.37

      ...

      2024-09-09 17:09:45.483 INFO tll.channel.input-channel: Recv message: type: Control, msgid: 10, name: Online, seq: 37, size: 0

      - После этого сообщения мы начинаем получать онлайн данные -

      ...                                                      

      2024-09-09 17:09:46.235 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 10, name: Transaction, seq: 38, size: 26
        time: 2024-09-09T14:09:46.235246986
        id: 39
        price: 510.24
        count: 93
      
      2024-09-09 17:09:46.236 INFO tll.channel.output-channel: Post message: type: Data, msgid: 20, name: Commission, seq: 38, size: 24
        time: 2024-09-09T14:09:46.235246986
        id: 39
        value: 474.52

      



