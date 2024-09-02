Глава 3. Сервис генерации случайных сделок
------------------------------------------

Нужно написать простую логику сервиса:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Сервис с определённой частотой генерирует сделки
  - Выходной поток - Случайная сделка (``Transaction``)
  - Задача состоит из 5 пунктов:

    - Генерация сообщений с фиксированной частотой
    - Генерация случайной сделки
    - Логика обработки сообщений ( C++ )
    - Конфиг процессора сервиса
    - Запуск и проверка работы
  - Рекомендую создать тестовую директорию у себя и уже в ней создавать файлы: ``$ mkdir gentest``, ``$ cd gentest``


|

Генерация сообщений с фиксированной частотой
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  
  - Наш сервис будет иметь специальный входной поток - ``timer``, он будет с заданной частотой генерировать входное сообщение с текущим временем
  - Сервис будет на каждое подобное сообщение генерировать сообщение со сделкой и отправлять его на выход
  - Инициализация потока:

    .. code:: yaml

      init:                           
      tll.proto: timer  # сообщаем, что поток - таймер                 
      timer:            # настройки таймера
        clock: realtime # генерировать сообщения с текущим временем системы
                        # если написать 'monotonic', то будут простые 'импульсы' без информации о текущем времени
        interval: 3s    # как часто генерировать сигнал
      dump: yes

Генерация случайной сделки
^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Сама сделка и процесс её генерации описан в файле ``transaction.h``:

    .. code:: c++

      #pragma once
      #include <random>                 // для генерации случайных данных          
      #include "tll/util/time.h"        // тип данных - time_point
      #include "tll/util/fixed_point.h" // тип данных - fixed_point
      
      class Transaction {
      public:

          // напоминаю, что у Transaction нами назначен msgid = 10
          static constexpr int MsgId = 10;
          
          // для более короткой записи типа данных - вещественное, с 2 знаками после запятой     
          using F2 = tll::util::FixedPoint<int64_t, 2>;
          
          // поля, аналогичные полям, которые описаны в файле 'transaction.yaml'
          tll::time_point time;
          int64_t id;
          F2 price;
          uint16_t count;
      public:

          // функция, для генерации случайной сделки с конкретным временем
          static Transaction GenerateRandom(tll::time_point tp) {
              Transaction tr;                   // создаём сделку
              tr.time = tp;                     // записываем время
              tr.id = _nextTransactionId++;     // записываем id, а затем увеличиваем его для будущих сделок
              tr.price = F2(_priceDistr(_gen)); // генерируем число, конвертируем его в F2, записываем как цену
              tr.count = _countDistr(_gen);     // генерируем число, записываем как количество
              return tr;
          }
      private:

          // id следующей сделки, нужно для генерации
          static int64_t _nextTransactionId;
          
          // объекты для генерации случайных чисел
          static std::random_device _rd;
          static std::mt19937 _gen;
          
          // случайные распределения цены и количества, с заданными границами
          static std::uniform_int_distribution<int64_t> _priceDistr;
          static std::uniform_int_distribution<uint16_t> _countDistr;
      };
      
      // первая сделка будет иметь id = 1
      int64_t Transaction::_nextTransactionId = 1;
      
      // создаём объекты
      std::random_device Transaction::_rd{ };
      std::mt19937 Transaction::_gen{ _rd() };
      
      // устанавливаем границы в конструкторе
      std::uniform_int_distribution<int64_t> Transaction::_priceDistr{ 1, 100000 }; // [0.01 - 1000.00]
      std::uniform_int_distribution<uint16_t> Transaction::_countDistr{ 1, 100 };

Логика обработки сообщений ( C++ )
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  
  - Файл, который описывает логику программы ``generator.cc``:

    .. code:: c++

      #include "tll/channel/module.h"       // нужно, чтобы объявить модуль, который затем использовать в '.yaml' файлах
      #include "tll/channel/tagged.h"       // класс, от которого мы будем наследоваться для упрощения реализации логики
      #include "tll/scheme/channel/timer.h" // кля обработки входного сообщения
      #include "transaction.h"              // кля генерации сделки
      
      // в файле 'tll/channel/tagged.h' описана логика, с помощью которой можно создавать каналы с различными именами
      // для простоты будущих реализаций там описаны 2 стандартных тэга: 
      using tll::channel::Input;
      using tll::channel::Output;
      // эти тэги позволяют работать с каналами 'input' и 'output', которые мы уже использовали, когда писали логику на питоне
      
      // в параметры шаблона передаётся текущий класс и все тэги, которые описывают обрабатываем каналы
      class Generator : public tll::channel::Tagged<Generator, Input, Output>
      {
      private:

          // в переменных будем хранить входной и выходной каналы
          tll::Channel * _input = nullptr;
          tll::Channel * _output = nullptr;
      public:

          // название нашего сервиса
          static constexpr std::string_view channel_protocol() { return "generator"; }
        
          // функция вызывается при создании сервиса
          // параметры аналогичны питоновским
          int _init(const tll::Channel::Url &, tll::Channel *master) {

              // получаем списки каналов
              auto & inputs = _channels.get<Input>();
              auto & outputs = _channels.get<Output>();
              
              // проверяем, что у нас ровно по одному каналу
              if (inputs.size() != 1)
                  return _log.fail(EINVAL, "Input size must be 1, got {}", inputs.size());
              if (outputs.size() != 1)
                  return _log.fail(EINVAL, "Output size must be 1, got {}", outputs.size());
                
              // сохраняем каналы в переменные
              // в inputs хранятся std::pair<>, first - указатель канала
              _input = inputs.begin()->first;
              _output = outputs.begin()->first; 
              
            return 0;
          }
      
          // данный метод вызывается при появлении сообщения в канале Input / 'input'
          int callback_tag(tll::channel::TaggedChannel<Input> * c, const tll_msg_t *msg) {

              // проверяем, что нам пришли именно данные
              if (msg->type != TLL_MESSAGE_DATA)
                  return 0;
            
              // проверяем, что канал соответсвует нужному
              if (c != _input)
                  return 0;
              
              // проверяем, что пришло сообщение с текущим временем
              if (msg->msgid != timer_scheme::absolute::id)
                  return 0;

              // получаем данные из этого сообщения
              // в структуре 'absolute' хранится единственное поле: 'ts' - текущее время
              auto timer = static_cast<const timer_scheme::absolute *>(msg->data);

              // создаём случайную сделку
              Transaction tr = Transaction::GenerateRandom(timer->ts);
                
              // создаём сообщение для отправки
              tll_msg_t transactionMsg = {
                  .type = TLL_MESSAGE_DATA,    // сообщение содержит данные
                  .msgid = Transaction::MsgId,
                  .seq = msg->seq,
                  .data = &tr,
                  .size = sizeof(tr)
              };
            
              // отправляем в выходной канал сообщение
              _output->post(&transactionMsg);
              return 0;
          }
          
          // данный метод вызывается при появлении сообщения в канале Output / 'output'
          int callback_tag(tll::channel::TaggedChannel<Output> * c, const tll_msg_t *msg) {   
              // ничего не делаем, потому что не ожидаем сообщений от выходного канала
              return 0; 
          }
      };
      
      // эти строки нужны для того, чтобы система сборки смогла данный класс преобразовать в модуль
      // этот модуль затем можно будет использовать в '.yaml' файлах для описания всего сервиса
      TLL_DEFINE_IMPL(Generator);
      TLL_DEFINE_MODULE(Generator);

  - Файл сборки, с помощью которого мы логику сформируем в модуль для дальнейшего использования ``meson.build``:

    .. code:: 

        project('generator', 'c', 'cpp'
          , version: '0.2.1'
          , default_options : ['cpp_std=c++17', 'werror=true', 'optimization=2']
          , meson_version: '>= 0.53'
          )

        fmt = dependency('fmt')
        tll = dependency('tll')
        
        shared_library('tll-generator'
          , ['generator.cc']
          , dependencies : [fmt, tll]
          , install: true
          )

  - Теперь мы должны собрать наш модуль, для этого используем команду: ``gentest$ meson build ; ninja -vC build``

Конфиг процессора сервиса
^^^^^^^^^^^^^^^^^^^^^^^^^

  - ``generator-processor.yaml``

    .. code:: yaml

      logger:
        type: spdlog 
        levels:
          tll: INFO
      
      processor.module:
        # название модуля, описанного в meson.build в shared_library(...)
        - module: build/tll-generator
      
      processor.objects:
        # входной поток - таймер
        input-channel:                     
          init:                           
            tll.proto: timer                 
            timer:
              clock: monotonic
              interval: 3s
            dump: yes
          depends: generator
      
        # будем записывать выходные данные в файл для проверки
        output-channel:              
          init:
            tll.proto: file             
            tll.host: output.dat       
            scheme: yaml://../comtest/transaction.yaml
            dir: w                          
            autoseq: true
            dump: scheme
            
        generator:
          # в init мы пишем строчку из channel_protocol() из класса Generator
          init: generator://
          channels: 
            input: input-channel
            output: output-channel
          depends: output-channel

Запуск и проверка работы
^^^^^^^^^^^^^^^^^^^^^^^^

  - Запускаем сервис: ``gentest$ tll-processor generator-processor.yaml``
  - Каждые 3 секунды мы будем видеть в логах подобное сообщение:

    .. code::

      2024-09-02 18:53:13.872 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 2, name: absolute, seq: 0, size: 8
        {ts: 2024-09-02T15:53:13.872665404}
      
      2024-09-02 18:53:13.872 INFO tll.channel.output-channel: Post message: type: Data, msgid: 10, name: Transaction, seq: 0, size: 32
        time: 2024-09-02T15:53:13.872665404
        id: 1
        price: 183.48
        count: 73
  - Проверим наш файл: ``gentest$ tll-read output.dat -seq 0:5``:

    .. code::

      - seq: 0
        name: Transaction
        data:
          time: '2024-09-02T15:57:53.492443617Z'
          id: 1
          price: '976.12'
          count: 69
      
      - seq: 1
        name: Transaction
        data:
          time: '2024-09-02T15:57:54.493915363Z'
          id: 2
          price: '769.11'
          count: 74
      ...

