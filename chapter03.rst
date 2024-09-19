Глава 3. Сервис генерации случайных сделок
------------------------------------------

Нужно написать простую логику сервиса:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Сервис с определённой частотой генерирует сделки
  - Выходной поток - случайная сделка (``Transaction``)
  - Задача состоит из 5 пунктов:

    - Генерация сообщений с фиксированной частотой
    - Генерация случайной сделки
    - Логика обработки сообщений ( C++ )
    - Конфиг процессора сервиса
    - Запуск и проверка работы


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

  - Для начала нам нужно сгенерировать С++ структуру из ``./messages/transaction.yaml`` файла, который был описан раньше, это можно сделать командой: ``tll-schemegen ./messages/transaction.yaml -o ./messages/transaction.h``
  - Немного отредактируем файл, чтобы он выглядел так, ``./messages/transaction.h``:

    .. code:: c++

      #pragma once
      #include <tll/scheme/types.h>
      #pragma pack(push, 1)

      namespace transaction_scheme{

      static constexpr std::string_view scheme_string = R"(yamls+gz://eJx9jbsKwzAMRfd+hTYtDTSlZPB3dC/GdkCQyMYPaAn598glyeBCN1107zkdsJ6dAnxGzUmbTJ7xAkBWQX+TYyQ32aTkAuhg2duZZodXyJ9QE3EeHhJ9qPOkYMHokp/KlyYFTvLF2sZ9/ApeVriuDZhsi20bIZL57z48I72dvf86jC+iPglFEP0gng1dylDf)";

      struct Transaction
      {
          static constexpr int msg_id = 10;
    
          std::chrono::time_point<std::chrono::system_clock, std::chrono::duration<int64_t, std::nano>> time;
          int64_t id;
          tll::scheme::FixedPoint<int64_t, 2> price;
          uint16_t count;
      };

      }; // namespace transaction_scheme
      #pragma pack(pop)


  - Сама сделка и процесс её генерации описан в файле ``./messages/transaction-generator.h``:

    .. code:: c++

      #pragma once
      #include <random>                   // для генерации случайных данных
      #include "transaction.h"            // только что созданный файл со структурой сообщения
      #include <tll/util/time.h>          // для tll::time_point
      
      class TransactionGenerator {
      private:
          using Transaction = transaction_scheme::Transaction;

          // id следующей сгенерированной сделки
          int64_t _next_transaction_id = 1;
          
          // объекты для генерации случайных чисел
          std::random_device _rd;
          std::mt19937 _gen;
          
          // равномерное распределение с границами, которые задаются в конструкторе
          std::uniform_int_distribution<int64_t> _price_distr;
          std::uniform_int_distribution<uint16_t> _count_distr;
      public:
          TransactionGenerator() 
              : 
              _gen{ _rd() }, 
              _price_distr{ 1, 100000 }, // [0.01 - 1000.00]
              _count_distr{ 1, 100 }
              {}
          
          Transaction generate_random_with_time( tll::time_point tp ) {
              Transaction tr;
              tr.time = tp;
              tr.id = _next_transaction_id++;

              // _price_distr(_gen) и _count_distr(_gen) возвращают
              // случайные целые числа из заданных промежутков
              tr.price = tll::util::FixedPoint<int64_t, 2> ( _price_distr(_gen) );
              tr.count = _count_distr(_gen);

              return tr;
          }
      };

Логика обработки сообщений ( C++ )
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  
  - Файл, который описывает логику программы ``generator.cc``:

    .. code:: c++

      // нужно, чтобы объявить модуль, который затем можно использовать в '.yaml' файлах
      #include <tll/channel/module.h>

      // класс, от которого мы будем наследоваться для упрощения реализации логики
      #include <tll/channel/tagged.h>  

      // для обработки входного сообщения     
      #include <tll/scheme/channel/timer.h> 

      // для генерации сделки
      #include "./messages/transaction-generator.h"              
      
      // в файле <tll/channel/tagged.h> описана вспомогательная логика
      // с помощью неё можно создавать потоки с различными именами
      // для простоты будущих реализаций там описаны 2 стандартных тэга: 
      using tll::channel::Input;
      using tll::channel::Output;
      // эти тэги позволяют работать с потоками 'input' и 'output'
      // их мы уже использовали, когда писали логику на питоне
      
      // в параметры шаблона передаётся текущий класс
      // и все тэги, которые описывают обрабатываемые потоки
      class Generator : public tll::channel::Tagged<Generator, Input, Output>
      {
      private:

          // в переменных будем хранить входной и выходной потоки
          tll::Channel * _input = nullptr;
          tll::Channel * _output = nullptr;
          TransactionGenerator _transaction_generator = {};
      public:

          // название нашего сервиса
          static constexpr std::string_view channel_protocol() { return "generator"; }
        
          // функция вызывается при создании сервиса
          // параметры аналогичны питоновским
          int _init(const tll::Channel::Url &, tll::Channel *master) {

              // проверяем, что у нас ровно по одному каналу, т.е. в промежутке [1, 1]
              if (check_channels_size<Input>(1, 1))
                  return EINVAL;
              if (check_channels_size<Output>(1, 1))
                  return EINVAL;

              // сохраняем потоки в переменные
              // в списках хранятся объекты std::pair<>, first - указатель канала
              _input = _channels.get<Input>().begin()->first;
              _output = _channels.get<Output>().begin()->first; 
              
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
              
              // получаем данные из этого сообщения
              // в структуре 'absolute' хранится единственное поле: 'ts' - текущее время
              auto timer = static_cast<const timer_scheme::absolute *>(msg->data);

              // создаём случайную сделку
              auto tr = _transaction_generator.generate_random_with_time(timer->ts);
                
              // создаём сообщение для отправки
              tll_msg_t transaction_msg = {
                  .type = TLL_MESSAGE_DATA,                          // сообщение содержит данные
                  .msgid = transaction_scheme::Transaction::msg_id,  // с нужным 'msgid'
                  .data = &tr,                                       // в 'data' хранится указатель на нужную структуру
                  .size = sizeof(tr)                                 // а в 'size' её размер
              };
            
              // отправляем в выходной канал сообщение
              _output->post(&transaction_msg);
              return 0;
          }
          
          // данный метод вызывается при появлении сообщения в канале Output / 'output'
          int callback_tag(tll::channel::TaggedChannel<Output> * c, const tll_msg_t *msg) {   
              // ничего не делаем, потому что не ожидаем сообщений от выходного канала (пока что!)
              return 0; 
          }
      };
      
      // эти строки нужны для того, чтобы система сборки смогла данный класс преобразовать в модуль
      // этот модуль затем можно будет использовать в '.yaml' файлах для описания всего сервиса
      TLL_DEFINE_IMPL(Generator);
      TLL_DEFINE_MODULE(Generator);

  - Файл сборки, с помощью которого мы логику сформируем в модуль для дальнейшего использования ``meson.build``:

    .. code:: meson

        # Название проекта может быть любое
        project('generator', 'c', 'cpp'
          , version: '0.2.1'
          , default_options : ['cpp_std=c++17', 'werror=true', 'optimization=2']
          , meson_version: '>= 0.53'
          )

        # Практически любой сервис будет зависеть от этих 2 библиотек
        # fmt - для логов ( форматирование правильное )
        # tll - для реализации логики сервиса
        fmt = dependency('fmt')
        tll = dependency('tll')
        
        # Создаём модуль с названием tll-generator
        shared_library('tll-generator'
          , ['generator.cc']          # Файл с классом логики
          , dependencies : [fmt, tll] # Зависимости
          , install: true
          )

  - Теперь мы должны собрать наш модуль, для этого используем команду: ``$ meson build ; ninja -vC build``

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
              clock: realtime
              interval: 3s
            dump: yes
          depends: generator
      
        # будем записывать выходные данные в файл для проверки
        output-channel:              
          init:
            tll.proto: file             
            tll.host: output.dat       
            scheme: yaml://./messages/transaction.yaml
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

  - Запускаем сервис: ``$ tll-processor generator-processor.yaml``
  - Каждые 3 секунды мы будем видеть в логах подобное сообщение:

    .. code::

      2024-09-02 18:53:13.872 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 2, name: absolute, seq: 0, size: 8
        {ts: 2024-09-02T15:53:13.872665404}
      
      2024-09-02 18:53:13.872 INFO tll.channel.output-channel: Post message: type: Data, msgid: 10, name: Transaction, seq: 0, size: 32
        time: 2024-09-02T15:53:13.872665404
        id: 1
        price: 183.48
        count: 73
  - Проверим наш файл: ``$ tll-read output.dat --seq 0:5``:

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
          time: '2024-09-02T15:57:56.493915363Z'
          id: 2
          price: '769.11'
          count: 74
      ...

