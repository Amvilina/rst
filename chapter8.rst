Глава 8. Знакомство со срезами
------------------------------




Что это такое и для чего нужны?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Срезы - удобный способ оптимизации работы сервисов-потребителей, которые работают с агрегированными данными
  - Предположим, что наш сервис-потребитель для своей корректной работы должен хранить некую агрегированную информацию приходящих сообщений ( минимум, сумма, разброс каких-то полей )
  - При запуске ( или переподключении ) сервис должен будет запросить у сервера все исторические данные и обработать их, это очень неэффективно !
  - Для этого сервер отдельно хранит специальные сообщения - срезы
  - Эти данные разделяются на группы, согласно бизнес-логике, в каждой из которых хранятся срезы с агрегированными данными
  - Каждый срез - сообщение с данными и с какими-то ``seq``. Если у среза ``seq=100``, это означает, что в нём хранится агрегированная информация обо всех сообщениях из промежутка ``[0 - 100]``

Срезы для клиента
^^^^^^^^^^^^^^^^^

  - Чтобы запросить определённый срез клиенту нужно в параметрах открытия канала ``request`` написать:

    .. code:: yaml

      open:
        mode: block           # ещё один режим открытия
        block: 0              # индекс среза с конца (0 = самый последний добавленный, 1 = предпоследний, ...)
        block-type: some-type # название группы, из которой хотим получить нужный срез

  - Срезы разделяются на группы, потому что у одного сервера могут быть много клиентов, каждому из которых нужны свои агрегированные данные для корректной работы
  - При открытии канала сервер отправляет клиенту сообщение с агрегированными данными. У этого сообщения есть определённый ``seq``. Сервер смотрит на него и 'переходит' в ``mode: seq-data`` со значением этого ``seq``, т.е. отправляет все исторические данные после среза
  - Клиент получает сообщение с агрегированными данными, затем начинает получать исторические данные, которые агрегирует самостоятельно, затем получает специальное сообщение ``Online`` и переходит к штатной работе

Вспомогательные классы
^^^^^^^^^^^^^^^^^^^^^^

  - Напишем несколько вспомогательных классов для нашей работы
  - Для начала сгенерируем C++ структуру сообщение ``Commission``: ``$ tll-schemegen ./messages/commission.yaml -o ./messages/commission.h``
  - После небольших исправлений файл выглядит так, ``./messages/commission.h``:

    .. code:: c++

      #pragma once
      #include <tll/scheme/types.h>
      #pragma pack(push, 1)

      namespace commission_scheme {

      static constexpr std::string_view scheme_string = R"(yamls+gz://eJx9jb0KwzAQg/c+hbZbGiihdPDaBykBX+DAf/Sc0hLy7j1DsnjIJqFP0oA0RXagZ45RVCUnugDiHcabiVk4eHWmgAHrDleJTFfUX2lOUn3czeZSra4OK71Zc1iaJQOSWkqNpr38KtlatG3dsPh+tic+U1jOv4+fWb7sx/bxB9sKRWc=)";

      struct Commission
      {
        static constexpr int msg_id = 20;
      
        std::chrono::time_point<std::chrono::system_clock, std::chrono::duration<int64_t, std::nano>> time;
        int64_t id;
        tll::scheme::FixedPoint<int64_t, 2> value;
      };
      
      } // namespace commission_scheme
      #pragma pack(pop)

  - Теперь напишем класс, который будет переводить ``Transaction`` в ``Commission``, ``commission-generator.h``:

    .. code:: c++

      #pragma once
      #include "commission.h"
      #include "transaction.h"
      
      class CommissionGenerator {
          using Transaction = transaction_scheme::Transaction;
          using Commission = commission_scheme::Commission;
          using F2 = tll::util::FixedPoint<int64_t, 2>;
      
      public:

          // На вход получаем транзакцию, а на выход отдаём комиссию
          static Commission create_from_transaction (const Transaction& transaction) {
              Commission commission;
              commission.time = transaction.time;
              commission.id = transaction.id;

              // метод .value() возвращает целое число, которое хранится на самом деле
              // F2(157.32).value() -> 15732
              commission.value = static_cast<F2> ( (transaction.price * transaction.count).value() / 100 );

              return commission;
          }
      };

  - Мы рассмотрим пример, когда клиент работает с агрегированными данными - сумма всех комиссий. Для простоты в срезе будем хранить сообщение ``Commission``, а данные агрегировать так: ``time=max(times), id=max(ids), value=sum(values)``
  - ``commission-agregator.h``:

    .. code:: c++

      #pragma once
      #include "commission.h"
      
      class CommissionAgregator {
      private:
          using Commission = commission_scheme::Commission;

          // храним внутри себя агрегированные данные
          Commission _commission;
      public:
          
          // возвращаем их по запросу
          Commission get( ) {
              return _commission;
          }
          
          // обновляем агрегированные данные
          void add ( const Commission& com ) {
              _commission.time = std::max(_commission.time, com.time); // максимум
              _commission.id = std::max(_commission.id, com.id);       // максимум
              _commission.value += com.value;                          // сумма
          }
      };
  - Для генерации среза серверу нужно отправить специальное сообщение ``Block``, которое он затем отправит каналу генерации срезов, само сообщение выглядит вот так, ``./messages/block.yaml``:

    .. code:: yaml

      - name: Block
        id: 100
        fields:
          # в поле хранится тип/группа среза
          # byte64 - массив из 64 байтов
          # string показывает, что стоит воспринимать каждый байт, как символ
          - {name: type, type: byte64, options.type: string}
  - И сгенерируем структуры для этого сообщения: ``$ tll-schemegen ./messages/block.yaml -o ./messages/block.h``, после небольшого рефакторинга файл выглядит так, ``./messages/block.h``:

    .. code:: c++

      #pragma once
      #include <tll/scheme/types.h>
      #pragma pack(push, 1)

      namespace block_scheme {

      static constexpr std::string_view scheme_string = R"(yamls+gz://eJzTVchLzE21UlB3yslPzlbnUlDITLFSMDQwALLSMlNzUoqtgCwFBV2FaqjCksqCVHUdBRAF5CVVlqSamQD5+QUlmfl5xVYK1RAVQLnikqLMvHT12louALk0HjA=)";

      struct Block
      {
        static constexpr int msg_id = 100;
      
        tll::scheme::ByteString<64> type;
      };
      
      } // namespace block_scheme
      #pragma pack(pop)

Канал генерации срезов
^^^^^^^^^^^^^^^^^^^^^^

  - Напишем на С++ новую логику, которая будет заниматься обработкой срезов для основного сервиса генерации сделок. На каждое входное сообщение ``Transaction`` система агрегирует данные, а при получении специального сообщения ``Block`` создаёт срез. При открытии канала на чтение ( в момент подключения через ``request`` клиента ) она находит нужный срез и возвращает его.
  - ``generator-block.cc``:

    .. code:: c++

      #include <tll/channel/module.h>
      #include <tll/channel/base.h>
      #include "./messages/commission-agregator.h"
      #include "./messages/commission-generator.h"
      #include "./messages/block.h"
      
      // создаём новый канал, наследуясь от базового класса
      class GeneratorBlock : public tll::channel::Base<GeneratorBlock> {
      private:

          // наш срез будем хранить наши данные в виде пары - {seq, commission}
          using BlockData = std::pair<long long, commission_scheme::Commission>;
          
          // строки, не вынесенные в константные переменные - моветон
          // в ней храним название группы срезов, которую мы можем обрабатывать
          const std::string BLOCK_TYPE_COMMISSION_SUM = "commission-sum";
      
          // храним агрегированную информацию
          CommissionAgregator _commission_agregator;
      
          // в векторе будут храниться все срезы
          std::vector<BlockData> _blocks;

          // seq последнего принятого сообщения
          long long _seq = -1;
      
          // в эту переменную мы будем записывать срез, который стоит отправить через 'request'
          BlockData _block_data;
      
          // нужно будет для корректной работы канала, при открытии его через 'request'
          GeneratorBlock * _master;
      
          // каналы работы с файлами
          std::unique_ptr<tll::Channel> _fileWrite;
          std::unique_ptr<tll::Channel> _fileRead;
      public:

          // мы будем сами управлять режимом работы функции process(...) из кода
          static constexpr auto process_policy() { return ProcessPolicy::Custom; }
          static constexpr std::string_view channel_protocol() { return "generator-block"; }
          
          int _init(const tll::Channel::Url &url, tll::Channel *master) {

              // строчка нужна, чтобы 'stream-server' заработал
              // она сообщает о схеме контрольного сообщения для работы с GeneratorBlock
              _scheme_control.reset(context().scheme_load(block_scheme::scheme_string));

              // конвертируем базовый класс в нужный нам и сохраняем
              if (master) 
                  _master = tll::channel_cast<GeneratorBlock>(master);
              return 0;
          }
      
      
          int _open(const tll::ConstConfig &cfg) {

              // Output = канал открыт для получения данных, он потребитель
              // в таком режиме открывает его 'stream-server', чтобы отправлять все данные
              if ( internal.caps & tll::caps::Output ) {
                  _create_and_open_files();            // создаём и открываем файлы
      
                  config_info().set_ptr("seq", &_seq); // связываем переменную с конфигом
                  config_info().setT("seq-begin", -1); // это значение будет использовано позже

                  // больше мы ничего не делаем при открытии в режиме Output
                  return 0;
              }
              
              // если верхний if не сработал, то мы в режиме tll::caps::Input
              // в таком режиме канал открывается через 'request', потому что данные он будет отдавать

              // считываем данные из параметров открытия в конфиге
              auto reader = tll::make_props_reader(cfg);
              auto block = reader.getT<unsigned>("block");
              auto type = reader.getT<std::string>("block-type", "default"); // после запятой можно указать значение по умолчанию
              if (!reader)
                  return _log.fail(EINVAL, "Invalid open parameters: {}", reader.error());
      
              // проверяем, что группа среза соответсвует нужной
              if (std::string(type) != BLOCK_TYPE_COMMISSION_SUM)
                  return _log.fail(EINVAL, "Unknown block type '{}', only '{}' supported", type, BLOCK_TYPE_COMMISSION_SUM);
              
        
              // когда мы создаём этот канал для 'request', то в master передаётся оригинальный канал
              // именно в этот оригинальный канал мы записывали данные, а он хранил их в _blocks
              // у нового канала свои переменные, поэтому мы должно обращаться именно к _master->_blocks

              // смотрим на сообщения с конца по индексу 'block' и записываем в переменную
              _block_data = _master->_blocks[_master->_blocks.size() - 1 - block];

              // записываем номер сообщения, указанный в срезе
              _seq = _block_data.first;

              // эти переменные использует 'stream-server' для корректной работы
              // 'seq-begin' - номер первого сообщения в блоке
              // 'seq' - номер последного сообщения в блоке
              // у нас всего 1 сообщение, поэтому они и совпадают
              config_info().setT("seq", _seq);
              config_info().setT("seq-begin", _seq);
      
              // обновляем dcaps канала
              // после этого у него будут постоянно вызываться функция process(...)
              // именно в ней нам нужно будет вызывать _callback(...), чтобы клиент получил данные
              _update_dcaps(tll::dcaps::Process | tll::dcaps::Pending);
              return 0;
          }
          
          int _process (long timeout, int flags) {

              // на всякий случай проверяем, что канал открыт именно на отправку данных
              if ( internal.caps & tll::caps::Output ) {
                  return 0;
              }
      
              // берём значение комиссии из переменной, в которую мы записали информацию в _open(...)
              auto commission = _block_data.second;
              
              // создаём сообщение с нужными параметрами
              tll_msg_t commissionMsg = {
                  .type = TLL_MESSAGE_DATA,
                  .msgid = commission_scheme::Commission::msg_id,
                  .seq = _block_data.first,
                  .data = &commission,
                  .size = sizeof(commission),
              };

              // и отправляем данные
              _callback(&commissionMsg);
              
              // так как у нас только 1 сообщение, то этот канал можно закрыть
              close();
              return 0;
          }
          
          // при закрытии канала мы убираем callback с конфига
          // а также закрываем канал, который работал с файлом на запись
          int _close() {
              config_info().setT("seq", _seq);
              _fileWrite->close();
              return Base::_close();
          }
          
          // 'stream-server' вызывает эту функцию и передаёт сюда каждое своё сообщение
          int _post(const tll_msg_t *msg, int flags) {

              // если это контрольное сообщение
              if (msg->type == TLL_MESSAGE_CONTROL) {

                  // проверяем, что это нужное нам контрольное сообщение
                  if (msg->msgid != block_scheme::Block::msg_id)
                      return _log.fail(EINVAL, "Invalid control message {}", msg->msgid);

                  // проверяем, что мы уже получали какие-то сообщения
                  if (_seq < 0)
                      return _log.fail(EINVAL, "Failed to make block: no data in storage");
      
                  // мы должны теперь создать новый срез

                  // конвертируем сообщение
                  auto block = *static_cast<const block_scheme::Block *>(msg->data);

                  // проверяем, что у него нужный block_type
                  if (std::string(block.type) != BLOCK_TYPE_COMMISSION_SUM)
                      return _log.fail(EINVAL, "Unknown block type '{}', only '{}' supported", block.type, BLOCK_TYPE_COMMISSION_SUM);
      
                  // создаём данные для среза
                  BlockData block_data = {_seq, _commission_agregator.get()};
      
                  // записываем срез в переменную
                  _create_block(block_data);

                  // записываем срез в память
                  _write_block_to_file(block_data);
                  return 0;
              }
              
              // если это сообщение с данными
              if (msg->type == TLL_MESSAGE_DATA) {

                  // то обрабатываем его в отдельной функции
                  _handle_input_data_msg(msg);
                  return 0;
              }
              
              return 0;
          }
      private:
          
          // функция создаёт и открывает каналы для работы с файлами
          void _create_and_open_files() {

              // это будет канал-файл, который будет обрабатывать 'block-storage.dat'
              auto curl = tll::ConfigUrl::parse("file://block-storage.dat");

              // название канала ( в логах, например, отображается оно )
              curl->set("name", "block-storage-write");

              // открываем файл на запись
              curl->set("dir", "w");

              // сообщаем о схеме сообщения, чтобы оно корректно отображалось при дебаге
              curl->set("scheme", "yaml://./messages/commission.yaml");

              // создаём канал с параметрами, описанными выше
              _fileWrite = context().channel(*curl, (tll::Channel *)this);

              // аналогично, но канал для чтения
              curl = tll::ConfigUrl::parse("file://block-storage.dat");
              curl->set("name", "block-storage-read");
              curl->set("dir", "r");
              _fileRead = context().channel(*curl, (tll::Channel *)this);

              // добавляем ему callback, который на каждое сообщение вызывает функцию _read_block_from_file(...)
              _fileRead->callback_add([](const tll_channel_t *c, const tll_msg_t *msg, void * user){
                  return static_cast<GeneratorBlock *>(user)->_read_block_from_file(msg);
              }, this, TLL_MESSAGE_MASK_ALL);

              // добавляем этот канал в process-loop основного (GeneratorBlock) канала
              // это нужно для того, чтобы у _fileRead вызывалась функция process(...)
              // именно в этой функции и происходит считываение данных
              _child_add(_fileRead.get(), "file-read");
      
              _fileWrite->open();
              _fileRead->open();
          }
      
          // эта функция будет вызываться каждый раз, когда у _fileRead есть сообщения для нас
          int _read_block_from_file(const tll_msg_t *msg)
          {
              // нас интересуют сообщения только с данными
              if (msg->type != TLL_MESSAGE_DATA)
                  return 0;
      
              // а сами сообщения должны быть 'Commission'
              if (msg->msgid != commission_scheme::Commission::msg_id) {
                  return _log.fail (EINVAL, "Unknown msgid: {}", msg->msgid);
              }
      
              // конвертируем сообщение в 'Commission' и сохраняем срез
              auto com = *static_cast<const commission_scheme::Commission*>(msg->data);
              _create_block({msg->seq, com});
      
              // обновляем _seq на самый большой ( последний ) встретившийся
              _seq = std::max(_seq, msg->seq);

              return 0;
          }
      
          // функция сохраняет срез в переменную
          void _create_block(BlockData block_data) {
              _blocks.push_back(block_data);
          }
      
          // функция записывает срез в память
          void _write_block_to_file(BlockData block_data) {
              tll_msg_t msg = {
                  .type = TLL_MESSAGE_DATA,  
                  .msgid = commission_scheme::Commission::msg_id,                        
                  .seq = block_data.first,
                  .data = &block_data.second,            
                  .size = sizeof(block_data.second)  
              };
      
              _fileWrite->post(&msg);
          }
          
          // функция вызывается при каждом входном сообщении с данными
          void _handle_input_data_msg(const tll_msg_t* msg) {

              // обновляем переменную, в которой храним 'seq' последнего сообщения
              _seq = msg->seq;

              // проверяем, что пришло сообщение 'Transaction' ( именно его stream-server генерирует и отправляет )
              if (msg->msgid != transaction_scheme::Transaction::msg_id)
                  return;

              // конвертируем сообщение в 'Transaction'
              auto transaction = *static_cast<const transaction_scheme::Transaction *>(msg->data);

              // создаём из него сообщение 'Commission'
              auto commission = CommissionGenerator::create_from_transaction(transaction);

              // отправляем данные агрегатору
              _commission_agregator.add(commission);
          }
          
      };
      
      // нужно для объявления модуля, чтобы потом его прописывать в процессоре
      TLL_DEFINE_IMPL(GeneratorBlock);
      TLL_DEFINE_MODULE(GeneratorBlock);

      

              
            

Соединение нового канала с сервисом
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Для начала нужно создать этот модуль в ``meson.build`` файле, ``meson.build``:

    .. code:: meson

      # ...

      shared_library('tll-generator-block'
        , ['generator-block.cc']    
        , dependencies : [fmt, tll] 
        , install: true
        )
  - После этого добавим несколько строк в ``generator-processor.yaml``:

    .. code:: yaml

      # ...

      processor.module:
      - module: build/tll-generator
      - module: build/tll-generator-block

      # ...

        output-channel:
          init:

            # ...

            block: generator-block://  # сообщаем, какой канал срезов использовать
            
            # если хранилище пустое, то добавить в него пустое сообщение Transaction при инициализации
            # это же сообщение автоматически отправляется и в канал block://
            init-message: Transaction  

            # если хранилище пустое, то после получения и обработки 'init-message'
            # отправить специальное сообщение 'Block' со значением 'type: commission-sum'
            init-block: commission-sum 

  - ``$ meson build --wipe; ninja -vC build``

Работа со срезами на стороне клиента
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    - КОД ПИТОНА


    - Для корректной работы ``msg.unpack()`` клиенту нужно знать о схемах всех сообщений. Для этого создадим специальный ``.yaml`` файл, куда включим всю информацию, ``./messages/all-schemes.yaml``:

      .. code:: yaml

        - import:
          - yaml://messages/commission.yaml
          - yaml://messages/transaction.yaml

    - Обновим конфиг процессора ``commission-processor.yaml``:

      .. code:: yaml

        # ...

        input-channel:
          init:

            # ...

            scheme: yaml://./messages/all-schemes.yaml

        # ...

Проверка работы
^^^^^^^^^^^^^^^
