Глава 5. Создание активного канала
----------------------------

  В предыдущих главах мы создали 2 сервиса, они представляли из себя логику (пассивный канал). Логика на каждое входное сообщение отправляла выходное, но сама ничего не генерировала. В этой главе снова рассмотрим процесс генерации случайных сделок, однако уже в виде активного канала

|

Коллбэки
^^^^^^^^

  - Каждый канал - наследник класса ``Base``, в этом классе определена специальная функция ``callback(...)``, которую можно вызвать, если мы хотим отправить данные
  - Другие логики, при использовании нашего канала (например, в роли ``Input``), имеют возможность добавить к ``callback(...)`` свою функцию-обработчик этих самых сообщений
  - В ``tll`` есть 2 стандартных автоматических способа: наследоваться от класса ``Logic``, а также наследоваться от класс ``Tagged`` (что мы уже делали). ``Tagged`` - новый и расширенный класс в сравнении с ``Logic``, он позволяет каждый канал представить в виде структуры со своими данными, которые могут быть необходимы для корректной его обработки 


Генерация сообщений
^^^^^^^^^^^^^^^^^^^

  - Создадим новый класс, для описания активного канала. ``generator-active.cc``:

    .. code:: c++

      #include <tll/channel/base.h>
      #include <tll/channel/module.h>
      #include "transaction-generator.h"
      
      class GeneratorActive : public tll::channel::Base<GeneratorActive> {
      private:
          TransactionGenerator _transactionGenerator = {};

      public:
          static constexpr std::string_view channel_protocol() { return "generator-active"; }

          // функция влияет на вызов функции _process
          // Normal - регулярно вызывается несколько раз в секунду
          // Never - никогда
          static constexpr auto process_policy() { return ProcessPolicy::Normal; }

          // та самая функция, которая регулярно вызывается
          // именно в ней мы будем генерировать сделки
          // timeout - максимальное время на обработку, чаще всего 0 == как можно быстрее
          // flags - пока не используется
          int _process(long timeout, int flags) {

              // генерируем сделку с текущим временем
              Transaction tr = _transactionGenerator.GenerateRandomWithTime(tll::time::now());
          
              // создаём сообщение
              tll_msg_t transactionMsg = {
                  .type = TLL_MESSAGE_DATA,                
                  .msgid = tll_message_info<Transaction>::id, 
                  .data = &tr,                                
                  .size = sizeof(tr)
              };

              // вызываем коллбэк, чтобы другие каналы могли его получить
              _callback(&transactionMsg);
              return 0;
          }
        
            
      };
      
      TLL_DEFINE_IMPL(GeneratorActive);
      TLL_DEFINE_MODULE(GeneratorActive);

  - Не забудем обновить ``meson.build``:

    .. code:: meson

      # ...

      shared_library('tll-generator-active'
        , ['generator-active.cc']
        , dependencies : [fmt, tll]
        , install: true
        )

      # ...

  - Собираем всё командой ``/gentest$ meson build; ninja -vC build``


Использование активного канала в сервисе комиссий
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Обновим наш ``commission-processor.yaml``:

    .. code:: yaml

      # ...

      # подключаём созданный нами модуль
      processor.module:
      - module: ../gentest/build/tll-generator-active

      processor.objects:
        input-channel:
          init:
            tll.proto: generator-active
            dump: scheme
            scheme: yaml://transaction.yaml
          depends: logic

        # ... output-channel ...

         logic:                             
          url: commission://              
          channels:            
            input: input-channel    # в момент инициализации логика commission добавит функцию-обработчик
                                    # каждый раз, когда generator-active будет вызывать callback(...)
                                    # в commission будет вызываться соответствующая фукнция     
            output: output-channel 
          depends: output-channel

  - Проверим, как это работает: ``/comtest$ tll-pyprocessor commission-processor.yaml``:

    .. code:: 

      2024-09-04 13:59:56.650 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 10, name: Transaction, seq: 0, size: 26
        time: 2024-09-04T10:59:56.649756169
        id: 1
        price: 532.87
        count: 27
      
      2024-09-04 13:59:56.651 INFO tll.channel.output-channel: Post message: type: Data, msgid: 20, name: Commission, seq: 0, size: 24
        time: 2024-09-04T10:59:56.649756169
        id: 1
        value: 143.87

      ...

  - Эти сообщения будут приходить со скоростью вызова функции ``_process(...)``

      
Задание частоты генерации сделок
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Наш канал успешно генерирует транзакции! Однако, это происходит очень быстро, нужно каким-то образом регулировать эту скорость. Для этого мы используем тот же принцип, что и с ``pub+tcp``, то есть префикс. В нашем случае подойдёт префикс ``rate+``
  - Обновим файл-конфиг ``commission-processor.yaml``:

    .. code:: yaml

      # ...

        input-channel:
          init:
            tll.proto: rate+generator-active
            rate:
              dir: in        # мы работает с input

              max-window: 1b # какой объем 'rate' пропустит без задержки 
                             # мы хотим сразу отправлять сообщения с заданной частотой
                             # поэтому устанавливаем минимальное значение

              speed: 26b     # если написать длину генерируемого сообщения, 
                             # то interval будет показывать частоту их генерации
                             # из логов в прошлом разделе видно, что Transaction - 26 byte
              interval: 3s
            dump: scheme
            scheme: yaml://transaction.yaml
          depends: logic

      # ...

  - Проверка ``/comtest$ tll-pyprocessor commission-processor.yaml``:

    .. code:: 

      2024-09-04 14:20:31.237 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 10, name: Transaction, seq: 0, size: 26
        time: 2024-09-04T11:20:31.237726279
        id: 1
        price: 499.37
        count: 27

      2024-09-04 14:20:31.238 INFO tll.channel.output-channel: Post message: type: Data, msgid: 20, name: Commission, seq: 0, size: 24
        time: 2024-09-04T11:20:31.237726279
        id: 1
        value: 134.83

      2024-09-04 14:20:34.238 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 10, name: Transaction, seq: 0, size: 26
        time: 2024-09-04T11:20:34.238364504
        id: 2
        price: 198.73
        count: 64

      2024-09-04 14:20:34.239 INFO tll.channel.output-channel: Post message: type: Data, msgid: 20, name: Commission, seq: 0, size: 24
        time: 2024-09-04T11:20:34.238364504
        id: 2
        value: 127.19

      ...

  - Сообщения приходят раз в 3 секунды, отлично !
