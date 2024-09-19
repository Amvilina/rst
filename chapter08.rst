Глава 8. Знакомство со срезами
------------------------------




Что это такое и для чего нужны?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  - Срезы - удобный способ оптимизации работы сервисов-потребителей, которые работают с агрегированными данными
  - Предположим, что наш сервис-потребитель для своей корректной работы должен хранить некую агрегированную информацию приходящих сообщений ( минимум, сумма, разброс каких-то полей )
  - При запуске ( или переподключении ) сервис должен будет запросить у сервера все исторические данные и обработать их, это очень неэффективно !
  - Для этого сервер отдельно хранит специальные сообщения - срезы
  - Эти данные разделяются на группы, согласно бизнес-логике, в каждой из которых хранятся срезы с агрегированными данными
  - Каждый срез - сообщения с данными. При отправке всех сообщений из среза сервер сообщает клиенту ``seq`` последнего сообщения, вошедшего в срез / агрегированные данные. Например, если клиенту сообщат ``seq=100``, то это значит, что в полученном срезе агрегированы сообщения с ``seq in [0 ... 100]``

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
      
      // для работы с файлами в директориях
      #include <filesystem>
      
      // создаём новый канал, наследуясь от базового класса
      class GeneratorBlock : public tll::channel::Base<GeneratorBlock> {
      private:
          using Commission = commission_scheme::Commission;
          using Transaction = transaction_scheme::Transaction;
      
          // наш срез будет хранить в себе:
          // seq - 'seq' последнего сообщения, которое учитывается в агрегированных данных
          // commission - агрегированные данные
          // сам срез вообще-то не обязан хранить 'seq', потому что срез - более атомарная единица
          // как клиент понимает, какой последний 'seq' был записан в агрегированные данные? - смотри место с 'seq-begin'
          // как в файлах хранится аналогичный 'seq'? - смотри функцию _read_block_from_file(...)
          struct BlockData {
              long long seq;
              Commission commission;
          };
      
          // строки, не вынесенные в константные переменные - моветон !!!
      
          // название группы срезов, которую мы можем обрабатывать
          const std::string BLOCK_TYPE_COMMISSION_SUM = "commission-sum";
      
          // название директории, где будут храниться срезы
          const std::string DIRECTORY = "blocks-storage";
      
          // хранить срезы будем в формате {FILE_PREFIX}.{index}.dat
          const std::string FILE_PREFIX = "block";
      
          // число сохранённых срезов = число файлов
          int _number_of_blocks = 0;
      
          // храним агрегированную информацию
          CommissionAgregator _commission_agregator;
      
          // seq последнего принятого сообщения
          long long _seq = -1;
      
          // будет использовано для корректной работы с данными в момент открытия канала на чтение
          GeneratorBlock * _master;
      
          // нужно для передачи информации из файла в код / из кода в канал 'request'
          BlockData _block;
      public:
      
          // мы будем сами управлять режимом работы функции process(...) из кода
          static constexpr auto process_policy() { return ProcessPolicy::Custom; }
      
          static constexpr std::string_view channel_protocol() { return "generator-block"; }
      
          int _init(const tll::Channel::Url &url, tll::Channel *master) {
              // в master будет приходить канал, в котором был создан этот канал
              // в нашем случае: при открытии GeneratorBlock на чтение ( т.е. для запроса через 'request' )
              // канал создаётся внутри канала GeneratorBlock, открытого на запись ( в него пишет stream-server )
              if (master)
                  // конвертируем базовый класс в нужный нам и сохраняем
                  _master = tll::channel_cast<GeneratorBlock>(master);
      
              // stream-server проверяет, что у канала есть схема со специальными сообщениями ( Block )
              // мы сообщаем, что у канала есть такая схема
              _scheme_control.reset(context().scheme_load(block_scheme::scheme_string));
              return 0;
          }
      
      
          int _open(const tll::ConstConfig &cfg) {
      
              // Output = канал открыт для получения данных, он потребитель
              // в таком режиме открывает его 'stream-server', чтобы записывать данные
              if ( internal.caps & tll::caps::Output ) 
                  return _handle_open_for_writer();
      
              
              // если верхний if не сработал, то мы в режиме tll::caps::Input
              // в таком режиме канал открывается через 'request', потому что из него будут читать данные
              return _handle_open_for_reader(cfg);   
          }
      
          // функция будет вызываться, пока мы отправляем данные на чтение
          // эта функция будет вызываться только после вызова функции _handle_open_for_reader(...)
          // в функции _handle_open_for_reader(...) в переменную _block записывается нужный нам срез
          int _process (long timeout, int flags) {
      
              // на всякий случай проверяем, что канал открыт именно на отправку данных
              if ( internal.caps & tll::caps::Output ) {
                  return 0;
              }
      
              // берём значения полей из переменной, в которую информация была записана при открытии канала
              tll_msg_t msg = {
                  .type = TLL_MESSAGE_DATA,
                  .msgid = Commission::msg_id,
                  .seq = _block.seq,
                  .data = &_block.commission,
                  .size = sizeof(_block.commission),
              };
      
              // отправляем данные ( через 'request' ) 
              _callback(&msg);
      
              // так как у нас только 1 сообщение, то этот канал можно закрыть
              close();
              return 0;
          }
      
          // при закрытии канала мы убираем callback с конфига
          int _close() {
              config_info().setT("seq", _seq);
              return Base::_close();
          }
      
          // 'stream-server' вызывает эту функцию и передаёт сюда каждое своё сообщение
          int _post(const tll_msg_t *msg, int flags) {
              
              // если это контрольное сообщение
              if (msg->type == TLL_MESSAGE_CONTROL)
                  return _handle_input_control_msg(msg);
      
              // если это сообщение с данными
              if (msg->type == TLL_MESSAGE_DATA)
                  return _handle_input_data_msg(msg);
      
              return 0;
          }
      private:
          // функция открывает канал на запись
          // это происходит один раз, для основной работы 'stream-server'
          int _handle_open_for_writer() {
      
              // сохраняем число всех сохранённых срезов
              _number_of_blocks = _get_number_of_blocks();
      
              // если сохранённые срезы есть
              if (_number_of_blocks > 0) {
      
                  // получаем самый последний срез
                  auto block = _get_block_by_index_from_last(0);
      
                  // и обновляем информацию
                  _seq = block.seq;
                  _commission_agregator.reset(block.commission);
              }
      
              // связываем переменную с конфигом
              config_info().set_ptr("seq", &_seq);
      
              return 0;
          }
      
          // функция возвращает число срезов, сохранённых в директории
          int _get_number_of_blocks() {
              int result = 0;
      
              // пробегаемся по всем файлам в директории
              for (auto & e : std::filesystem::directory_iterator { DIRECTORY }) {
      
                  // получаем имя файла
                  auto filename = e.path().filename();
      
                  // .stem() -> возвращает значение до последней точки ( без расширения )
                  // здесь мы проверяем, что файлы имеют нужное название
                  if (filename.stem().stem() != FILE_PREFIX) // {FILE_PREFIX}.index.dat -> {FILE_PREFIX}
                      continue;
      
                  // .extenstion() -> возвращает последнюю точку и всё после неё ( расширение )
                  // здесь проверяем, что имеют нужное расширение
                  if (filename.extension() != ".dat")
                        continue;
                        
                  // считаем число нужных файлов
                  ++result;
              }
              return result;
          }
      
          // функция возвращает срез по индексу
          // индексы здесь с конца ( 0 - самый последний срез, 1 - предпоследний, ... ) 
          BlockData _get_block_by_index_from_last(int index) {
              
              // получаем путь к файлу по данному индексу
              auto path = _get_path_for_block_by_index_from_last(index);
      
              // создаём конфиг для создания канала
              auto curl = tll::ConfigUrl::parse("file://");
      
              // файл будет открыт для чтения
              curl->set("dir", "r");
      
              // создаём канал, который будет читать файл
              auto file = context().channel(*curl, (tll::Channel *)this);
      
              // добавляем коллбэк 
              // на каждое сообщение из файла будет вызываться он -> вызываться функция _read_block_from_file(...)
              file->callback_add([](const tll_channel_t *c, const tll_msg_t *msg, void * user){
                  return static_cast<GeneratorBlock *>(user)->_read_block_from_file(msg);
              }, this, TLL_MESSAGE_MASK_ALL);
      
              // создаём конфиг на открытие канала
              auto open_config = tll::Config();
      
              // указываем ему нужный путь к файлу
              open_config.set("filename", path);
      
              // открываем файл и проверяем, что получилось открыть
              file->open(open_config);
              if (file->state() != tll::state::Active)
                  return _log.fail(BlockData{}, "Can not open file {}", path);
      
              // в функции process(...) происходит чтение данных из файла
              // это плохой пример! на практике не стоит вызывать функцию непосредственно
              // стоит добавлять канал file в process основного канала, чтобы эта функция автоматически вызывалась
              // мы знаем, что в каждом файле будет храниться ровно 1 сообщение, поэтому можем себе позволить :)
              file->process();
      
              // после вызова функции _read_block_from_file(...) информация записывается в переменную _block
              return _block;
          }
      
          // функция находит путь к файлу по индексу
          std::string _get_path_for_block_by_index_from_last(int index) {
      
              // мы будем здесь использовать max-heap
              std::vector<int> indexes;
      
              // пробегаемся по каждому файлу
              for (auto & e : std::filesystem::directory_iterator { DIRECTORY }) {
      
                  // аналогичные проверки
                  auto filename = e.path().filename();
                  if (filename.stem().stem() != FILE_PREFIX)
                      continue;
                  if (filename.extension() != ".dat")
                        continue;
      
                  // считываем индекс из имени файла {FILE_PREFIX}.{seq}.dat
                  // filename = {FILE_PREFIX}.{seq}.dat
                  // .stem() -> {FILE_PREFIX}.{seq}
                  // .extension() -> .{seq}
                  // string().substr(1) -> {seq}
                  // std::stoi(...) переводит строку в число
                  int ind = std::stoi(filename.stem().extension().string().substr(1));
                  indexes.push_back(ind);
              }
      
              // создаём max-heap из наших индексов
              std::make_heap(indexes.begin(), indexes.end());
      
              // удаляем 'index' максимальных индексов
              for (int i = 0; i < index; ++i) {
                  std::pop_heap(indexes.begin(), indexes.end());
                  indexes.pop_back();
              }
      
              // сохраняем максимальный индекс из оставшихся
              // именно он и будет иметь 'index' с конца всех
              std::pop_heap(indexes.begin(), indexes.end());
              int ind = indexes.back();
      
              // возвращаем корректный путь
              return DIRECTORY + "/" + FILE_PREFIX + "." + std::to_string(ind) + ".dat" ;
          }
      
          // функция, которая вызывается на каждое считанное сообщение ( и управляющие сообщения ) из файла
          int _read_block_from_file(const tll_msg_t* msg) {
      
              // нам нужны только данные из файла
              if (msg->type != TLL_MESSAGE_DATA)
                  return 0;
      
              // проверяем, что в файле хранится именно Commission
              if (msg->msgid != Commission::msg_id) {
                  return _log.fail (EINVAL, "Unknown msgid: {}", msg->msgid);
              }
      
              // конвертируем сообщение в 'Commission' и сохраняем срез
              _block.commission = *static_cast<const Commission*>(msg->data);
      
              // в нашей задаче мы храним в файле Commission, у которого 'seq' = 'seq' всего среза
              // не всегда так стоит делать, но в нашей задаче это удобно, потому что наш срез всего состоит из 1-го сообщения
              _block.seq = msg->seq;
              
              return 0;
          }
      
          // функция открывает канал на чтение
          // это происходит каждый раз, когда приходит запрос через 'request'
          int _handle_open_for_reader(const tll::ConstConfig &cfg) {
      
              // считываем пришедшие параметры
              auto reader = tll::make_props_reader(cfg);
              auto block_index = reader.getT<int>("block");
              auto type = reader.getT<std::string>("block-type", "default"); // после запятой можно указать значение по умолчанию
      
              // проверяем их на корректность
              if (!reader)
                  return _log.fail(EINVAL, "Invalid open parameters: {}", reader.error());
              
              // здесь мы используем _master, потому что только у GeneratorBlock-писателя хранится информация о числе файлов
              if (block_index < 0 || block_index > _master->_number_of_blocks - 1)
                  return _log.fail(EINVAL, "Block number: {} out of bounds: [{} - {}]", block_index, 0, _number_of_blocks - 1);
      
              // проверяем, что группа среза соответсвует нужной
              if (std::string(type) != BLOCK_TYPE_COMMISSION_SUM)
                  return _log.fail(EINVAL, "Unknown block type '{}', only '{}' supported", type, BLOCK_TYPE_COMMISSION_SUM);
      
              // получаем нужный срез
              auto block = _get_block_by_index_from_last(block_index);
              _seq = block.seq;
      
              // эта переменная будет использована в функции _process(...) для отправки данных
              _block = block;
      
              // эти переменные использует 'stream-server' для корректной работы
              // 'seq-begin' - номер первого сообщения в блоке
              // 'seq' - номер последного сообщения в блоке
              // 'stream-client' после получения блока начнёт получать сообщения с 'seq' = "seq" + 1, "seq" + 2, ... 
              // у нас всего 1 сообщение, поэтому они и совпадают
              config_info().setT("seq", _seq);
              config_info().setT("seq-begin", _seq);
      
              // обновляем dcaps канала
              // после этого у него будут постоянно вызываться функция process(...)
              // именно в ней нам нужно будет вызывать _callback(...), чтобы клиент получил данные
              _update_dcaps(tll::dcaps::Process | tll::dcaps::Pending);
              return 0;
          }
      
          // обработка контрольных сообщений
          int _handle_input_control_msg(const tll_msg_t* msg) {
      
              // у нас есть только одно контрольное сообщение
              if (msg->msgid != block_scheme::Block::msg_id)
                  return _log.fail(EINVAL, "Invalid control message {}", msg->msgid);
      
              // проверяем, что мы уже получали какие-то сообщения
              if (_seq < 0)
                  return _log.fail(EINVAL, "Failed to make block: no data in storage");
      
      
              // конвертируем сообщение
              auto block = *static_cast<const block_scheme::Block *>(msg->data);
      
              // проверяем, что у него нужный block_type
              if (std::string(block.type) != BLOCK_TYPE_COMMISSION_SUM)
                  return _log.fail(EINVAL, "Unknown block type '{}', only '{}' supported", block.type, BLOCK_TYPE_COMMISSION_SUM);
      
              // получаем агрегированную информацию
              auto commission_agregated = _commission_agregator.get();
      
              // создаём сообщение для записи
              // в 'seq' будет храниться 'seq' последнего вошедшего в агрегированные данные сообщения
              tll_msg_t post_msg = {
                  .type = TLL_MESSAGE_DATA,
                  .msgid = Commission::msg_id,
                  .seq = _seq,
                  .data = &commission_agregated,
                  .size = sizeof(commission_agregated)
              };
              
              // имя нового файла ( индекс на 1 больше последнего )
              std::string path = DIRECTORY + "/" + FILE_PREFIX + "." + std::to_string(_number_of_blocks+1) + ".dat";
      
              // создаём канал для записи в файл
              auto curl = tll::ConfigUrl::parse("file://");
              curl->set("dir", "w");
      
              // нужно для корректного дебагинга
              curl->set("scheme", "yaml://./messages/commission.yaml");
      
              auto file = context().channel(*curl, (tll::Channel *)this);
      
              // создаёем конфиг на открытие нужного файла и открываем его
              auto open_config = tll::Config();
              open_config.set("filename", path);
              file->open(open_config);
      
              // проверяем, что успешно открыли
              if (file->state() != tll::state::Active)
                  return _log.fail(EINVAL, "Can not open file {}", path);
      
              // записываем в него сообщение
              file->post(&post_msg);
      
              // увеличиваем число записанных срезов
              ++_number_of_blocks;
      
              return 0;
          }
      
          // обработка сообщений с данынми
          int _handle_input_data_msg(const tll_msg_t* msg) {
      
              // обновляем переменную, в которой храним 'seq' последнего сообщения
              _seq = msg->seq;
      
              // проверяем, что пришло сообщение 'Transaction' ( именно его 'stream-server' генерирует и отправляет )
              if (msg->msgid != Transaction::msg_id)
                  return _log.fail(EINVAL, "unnknown msgid for blocks channel: {}", msg->msgid);
      
              // конвертируем сообщение в 'Transaction'
              auto transaction = *static_cast<const Transaction *>(msg->data);
      
              // создаём из него сообщение 'Commission'
              auto commission = CommissionGenerator::create_from_transaction(transaction);
      
              // отправляем данные агрегатору
              _commission_agregator.add(commission);
      
              return 0;
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
    - Добавим обработку входящего среза у нашего клиента, ``commission.py``:

      .. code:: python

        # ...

            def _init(self, url, master=None):

                # ...

                request_config["mode"] = 'block'
                request_config["block"] = '0'
                request_config["block-type"] = 'commission-sum'

                # в переменной будет храниться сумма всех комиссий
                self._commission_sum = 0.0 

        # ...

            def _logic(self, channel, msg):
                if channel != self._input:
                    return
                if msg.type != msg.Type.Data:
                    return
                msg = channel.unpack(msg)
      
                if msg.SCHEME.name == 'Transaction':
                  value = msg.price * msg.count * decimal.Decimal('0.01')

                  # обновляем сумму с каждой новой рассчитанной комиссией
                  self._commission_sum += value
      
                  self._output.post(
                      {'time': msg.time, 'id': msg.id, 'value': value},
                      name='Commission')
              
              # если пришло сообщение 'Commission', то мы получили срез
              if msg.SCHEME.name == 'Commission':

                  # обновляем значение суммы на значение среза
                  self._commission_sum = msg.value


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

    - Для начала удалим старые данные из нашего хранилица: ``rm -r storage/`` ( в предыдущей главе было описано ). После этого создадим нужные директории: ``mkdir storage; mkdir blocks-storage``
    - Запустим сначала сервер: ``tll-processor generator-processor.yaml``, а затем через какое-то время в другом окне терминала клиента: ``tll-pyprocessor commission-processor.yaml``
    - В логах клиента можно увидеть:

      .. code::

        2024-09-16 19:31:20.433 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 20, name: Commission, seq: 0, size: 24
          time: 1970-01-01T00:00:00
          id: 0
          value: 0.00
        
        2024-09-16 19:31:20.433 INFO tll.channel.input-channel: Recv message: type: Data, msgid: 10, name: Transaction, seq: 1, size: 26
          time: 2024-09-16T16:31:17.313670847
          id: 1
          price: 604.77
          count: 55

        ...

        2024-09-16 19:31:20.434 INFO tll.channel.input-channel: Reached reported server seq 4, no online data
        2024-09-16 19:31:20.434 INFO tll.channel.input-channel: Stream is online on seq 4
        2024-09-16 19:31:20.434 INFO tll.channel.input-channel: Recv message: type: Control, msgid: 10, name: Online, seq: 4, size: 0

        ...
    - В хранилище можем увидеть наш пустой срез: ``tll-read blocks-storage/block.1.dat``

      .. code::

        - seq: 0
          name: Commission
          data:
            time: '1970-01-01T00:00:00Z'
            id: 0
            value: '0.00'



