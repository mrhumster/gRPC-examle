https://grpc.io/docs/languages/python/basics/

# Зачем использовать gRPC

Наш пример - простое приложение для составления карты маршрута, которое позволяет клиентам получать информацию об объектах на своем маршруте, создавая сводку своего маршрута и обмениваться информацией о маршруте, такой как обновление трафика, с сервером и другими клиентами.

С помощью gRPC мы можем определить нашу службу один раз в `.proto` файле и генерировать клиентов и серверы на любом из поддерживаемых gRPC языков, которые, в свою очередь, могут быть запущены в средах от серверов внутри большого центра обработки данных до вашего собственного планшета - вся сложность коммуникации между различными языками и средами обрабатывает для вас gRPC. Мы также получаем все преимущества работы с буферами протоколов, включая эффективную сериализацию, простой IDL и простое обновление интерфейса.

# Определение услуги

Первый шаг - определение *службы* gRPC и типов *запросов* и *ответов* методов с использованием protobuf. 

Чтобы определить службу, укажите её имя `service` в `.proto` файле.

```protobuf
service RouteGuide {
    // (Method definitions not show)
}
```

Затем вы определяете `rpc` методы внутри определения вашей службы, указывая их типы запросов и ответов. gRPC позволяет вам определить четыре вида методов службы, все из которых используются в `RouteGuide` службе:

* Простой *RPC*, при котором клиент отправляет запрос на сервер с помощью заглушки и ждёт ответа, как при обычном вызове функции.

```protobuf
// Obtains the feature at a given position.
rpc GetFeature(Point) returns (Feature) {}
```

* RPC *ответ-поток*, где клиент отправляет запрос на сервер и получает поток для чтения последовательности сообщений. Клиент читает из возвращенного потока до тех пор, пока не останется больше сообщений. Как вы можете видеть в примере, вы указываете метод ответа-потока, помещая ключевое слово `stream` перед типом *ответа*.

```protobuf
// Obtains the Features available within the given Rectangle. Results are 
// streamed rather than returned at once (e.g. in a response message with a 
// repeated field), as the rectangle may cover a large area and contain a 
// huge number of features.
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

* RPC *запрос-поток*, где клиент пишет последовательность сообщений и отправляет их на сервер, снова используется предоставленный поток. После того как клиент закончит писать сообщения, он ждёт, пока сервер прочитает их все и вернет свой ответ. Вы указываете метод запроса-потока, помещая ключевое слово `stream` перед типом *запроса*.

```protobuf
// Acceprt a stream of Points on a route being traversed, returning a
// RouteSummary when traversal is completed.
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

* Двунаправленный *потоковый* RPC, где обе стороны отправляют последовательность сообщений с помощью потока чтения-записи. Два потока работают независимо, поэтому клиенты и серверы могуть читать и писать в любом порядке: например, сервер может ждать получения всех сообщений клиента, прежде чем писать свои ответы, или он может поочередно читать сообщение, а затем писать сообщение, или выполнять какую-либор другую комбинацию чтения и записи. Порядок сообщений в каждом потоке сохраняется. Вы указыавете этот тип метода, помещая ключевое слово `stream` перед запросом и ответом.

```protobuf
// Accept a tream of RouteNotes sent while a route is being traversed,
// while receiving other RouteNotes (e.g. from other users).
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

Ваш `.proto` файл также содержит определения типов сообщений буфера протокола для всех типов запросов и ответов, используемых в наших методах обслуживания, — например, вот `Point`.

```protobuf
// Points are represented as latitude-longitude pairs in the E7 representaion
// (degrees multioplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
    int32 latitude = 1;
    int32 longitude = 2;
}
```

# Генерация клиентского и серверного кода

Далее вам необходимо сгенерировать клиентские и серверные интерфейсы gRPC из `.proto` определения вашей службы.

```bash
# Сначала установите пакеты
$ pip install grpcio-tools
```

```bash
# Для генерации кода Python используйте следующую команду
~/PycharmProjects/gRPC-examle/src/route (master*) » python -m grpc_tools.protoc -I../protos --python_out=. --pyi_out=. --grpc_python_out=. ../protos/route.proto 
```

```bash
~/PycharmProjects/gRPC-examle/src (master*) » tree                                           
.
├── protos
│   └── route.proto
└── route
    ├── route_pb2_grpc.py
    ├── route_pb2.py
    └── route_pb2.pyi

3 directories, 4 files

```

Сгенерированные файлы кода называются `route_pb2.py` и `route_pb2_grpc.py` содержат:

* Классы для сообщений, определённые в `route.proto`.
* Классы для служб, определённые в `route.proto`.
    * `RouteStub`, который может использоваться клиентами для вызова RPC RouteGuide.
    * `RouteServicer`,  который определяет интерфейс для реализации сервиса RouteGuide.
* Функции для службы, определенной в `route.proto`, который добавляет RouteServicer к `grpc.Server`.

# Создание сервера

Сначала давайте рассмотрим, как создать `Route` сервер. 

Создание и запуск сервера делится на два этапа работы:

* Реализация интерфейса, созданного на основе определения нашего сервиса, с функциями, которые выполняют фактическую работу сервиса.
* Запуск сервера gRPC для прослушивания запросов от клиентов и передачи ответов.

## Реализация Route

`route_server_py` содержит класс `RouteGuideServicer`, который является подклассом сгенерированного класса `route_pb2_grpc.RouteGuideServicer`:

```python
class RouteGuideServicer(route_pb2_grpc.RouteGuideServicer):
```

`RouteGuideServicer` реализует все `RouteGuide` методы обслуживания.

### Простой RPC

Давайте сначала рассмотрим простейший тип, `GetFeature` который просто получает `Point` от клиента и возвращает соответсвующую информацию о функциях из своей базы данных в виде `Feature`.

```python
def GetFeature(self, request, context):
    feature = get_feature(self.db, request)
    if feature in None:
        return route_guide_pb2.Feature(name="", location=request)
    else:
        return feature
```

### Потоковая передача ответов RPC

Теперь давайте рассмотрим следующий метод. `ListFeatures` это потоковый RPC-ответ, который отправляет клиенту `Feature` несколько сообщений.

```python
def ListFeatures(self, request, context):  
    left = min(request.lo.longitude, request.hi.longitude)  
    right = max(request.lo.longitude, request.hi.longitude)  
    top = max(request.lo.latitude, request.hi.latitude)  
    bottom = min(request.lo.latitude, request.hi.latitude)  
    for feature in self.db:  
        if (  
                left <= feature.location.longitude <= right  
                and bottom <= feature.location.latitude <= top  
        ):  
            yield feature
```

### Запрос-поток RPC

Метод потоковой передачи запросов `RecordRoute` использует итератор значений запрос и возвращает одно значение ответ.

```python
def RecordRoute(self, request_iterator, context):  
    point_count = 0  
    feature_count = 0  
    distance = 0.0  
    prev_point = None  
  
    start_time = time.time()  
    for point in request_iterator:  
        point_count += 1  
        if get_feature(self.db, point):  
            feature_count += 1  
        if prev_point:  
            distance += get_distance(prev_point, point)  
        prev_point = point  
  
    elapsed_time = time.time() - start_time  
    return route_pb2.RouteSummary(  
        point_count=point_count,  
        feature_count=feature_count,  
        distance=int(distance),  
        elapsed_time=int(elapsed_time),  
    )
```

### Двунаправленный потоковый RPC

Наконец, давайте рассмотрим метод двунаправленной потоковой передачи `RouteChat`.

```python
def RouteChat(self, request_iterator, context):  
    prev_notes = []  
    for new_note in request_iterator:  
        for prev_note in prev_notes:  
            if prev_note.location == new_note.location:  
                yield prev_note  
        prev_notes.append(new_note)
```

Семантика этого метода представляет собой комбинацию семантики метода `request-streaming` и метода `response-streaming`. Он передает итератор значений запроса и сам является итератором значений ответа.

## Запуск сервера

После реализации всех методов `RouteGuide`, следующим шагом станет запуск сервера gRPC, что бы клиенты могли использовать ваш сервис:

```python
def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))  
    # Создаётся gRPC сервер с использованием пула потоков. 
    # futures.ThreadPoolExecutor(max_workers=10) означает, что сервер 
    # будет обрабатывать запросы с помощью до 10 потоков одновременно.

    route_pb2_grpc.add_RouteGuideServicer_to_server(RouteGuideServicer(), server)  
    # Добавление реализации сервиса (RouteGuideServicer) к gRPC серверу.
    # Эта строка регистрирует наш сервис, чтобы сервер знал, как обрабатывать запросы.

    server.add_insecure_port("[::]:50051")  
    # Открытие небезопасного порта (без TLS). Здесь используется IPv6 адрес 
    # "[::]" и порт 50051 для приёма входящих запросов.

    server.start()  
    # Запуск сервера. С этого момента сервер начинает слушать запросы на порту.

    server.wait_for_termination()  
    # Сервер ожидает завершения работы, удерживая процесс запущенным до тех пор, 
    # пока сервер не будет остановлен вручную или аварийно завершён.

```

Метод сервера `start()` не блокируется. Будет создан новый поток для обработки запросов. Вызывающий поток `server.start()` часто не будет иметь никакой другой работы в это время. В этом случае вы можете вызывать `server.wait_for_termination()` чтобы чисто заблокировать вызывающий поток, пока сервер не завершит работу.

# Создание клиента

## Создание заглушки

Для вызова методов служб нам сначала нужно создать *заглушку*.

Мы создаем экземпляр `RouteGuideStub` класс модуля `route_pb2_grpc`, сгенерированного из нашего `.proto`.

```python
channel = grpc.insecure_channel('localhost:50051')
stub = route_guide_pb2_grpc.RouteGuideStub(channel)
```

## Методы вызова службы

Для методов RPC, возвращающих один ответ, gRPC Python поддерживает как синхронную (блокирующую), так и асинхронную (неблокирующую) семантику потока управления. Для методов RPC с потоком ответов вызовы немедленно возвращают итератор значений ответов. Вызов метода `next()` этого итератора блокируется до тех пор, пока ответ, который должен быть получен от итератора, не станет доступным.

### Простой RPC

Синхронный вызов простого RPC `GetFeature` почти так же прост, как вызов локального метода. Вызов RPC ждет ответа сервера и либо вернет ответ, либо вызовет исключение:

```python
feature = stub.GetFeature(point)
```

Асинхронный вызов `GetFeature` поход на асинхронный вызов локального метода в пуле потоков:

```python
feature_future = stub.GetFeature.future(point)
feature = feature_future.result()
```

### Потоковая передача ответов RPC

Вызов response-streaming `ListFeatures` аналогичен работе с типами последовательностей:

```python
for feature in stub.ListFeatures(rectangle):
```

### Запрос-поток RPC

Вызов request-streaming `RecordRoute` похож на передачу итератора локальному методу. Как и простой RPC выше, который также возвращает один ответ, его можно вызвать синхронно или асинхронно.:

```python
route_summary = stub.RecordRoute(point_iterator)
```

```python
route_summary_future = stub.RecordRoute.future(point_iterator)
route_summary = route_summary_future.result()
```

### Двунаправленный потоковый RPC

Вызов двунаправленной потоковой передачи `RouteChat` (как и в случае со стороны сервиса) представляет собой комбинацию семантики потоковой передачи запроса и потоковой передачи ответа:

```python
for received_route_note in stub.RouteChat(sent_route_note_iterator):
```

# Попробуйте!

Запускаем сервер:

```bash
$ python route_server.py
```

Запустите клиент с другого терминала:

```bash
$ python route_client.py
```

```bash
$ python route_client.py                           
-------------- GetFeature --------------
Feature called 'Berkshire Valley Management Area Trail, Jefferson, NJ, USA' at latitude: 409146138, longitude: -746188906
Found no feature at latitude: 0, longitude: 0
-------------- ListFeatures --------------
Looking for features between 40, -75 and 42, -73
Feature called 'Patriots Path, Mendham, NJ 07945, USA' at latitude: 407838351, longitude: -746143763
Feature called '101 New Jersey 10, Whippany, NJ 07981, USA' at latitude: 408122808, longitude: -743999179
Feature called 'U.S. 6, Shohola, PA 18458, USA' at latitude: 413628156, longitude: -749015468
Feature called '5 Conners Road, Kingston, NY 12401, USA' at latitude: 419999544, longitude: -740371136
Feature called 'Mid Hudson Psychiatric Center, New Hampton, NY 10958, USA' at latitude: 414008389, longitude: -743951297
Feature called '287 Flugertown Road, Livingston Manor, NY 12758, USA' at latitude: 419611318, longitude: -746524769
Feature called '4001 Tremley Point Road, Linden, NJ 07036, USA' at latitude: 406109563, longitude: -742186778
Feature called '352 South Mountain Road, Wallkill, NY 12589, USA' at latitude: 416802456, longitude: -742370183
Feature called 'Bailey Turn Road, Harriman, NY 10926, USA' at latitude: 412950425, longitude: -741077389
Feature called '193-199 Wawayanda Road, Hewitt, NJ 07421, USA' at latitude: 412144655, longitude: -743949739
Feature called '406-496 Ward Avenue, Pine Bush, NY 12566, USA' at latitude: 415736605, longitude: -742847522
Feature called '162 Merrill Road, Highland Mills, NY 10930, USA' at latitude: 413843930, longitude: -740501726
Feature called 'Clinton Road, West Milford, NJ 07480, USA' at latitude: 410873075, longitude: -744459023
Feature called '16 Old Brook Lane, Warwick, NY 10990, USA' at latitude: 412346009, longitude: -744026814
Feature called '3 Drake Lane, Pennington, NJ 08534, USA' at latitude: 402948455, longitude: -747903913
Feature called '6324 8th Avenue, Brooklyn, NY 11220, USA' at latitude: 406337092, longitude: -740122226
Feature called '1 Merck Access Road, Whitehouse Station, NJ 08889, USA' at latitude: 406421967, longitude: -747727624
Feature called '78-98 Schalck Road, Narrowsburg, NY 12764, USA' at latitude: 416318082, longitude: -749677716
Feature called '282 Lakeview Drive Road, Highland Lake, NY 12743, USA' at latitude: 415301720, longitude: -748416257
Feature called '330 Evelyn Avenue, Hamilton Township, NJ 08619, USA' at latitude: 402647019, longitude: -747071791
Feature called 'New York State Reference Route 987E, Southfields, NY 10975, USA' at latitude: 412567807, longitude: -741058078
Feature called '103-271 Tempaloni Road, Ellenville, NY 12428, USA' at latitude: 416855156, longitude: -744420597
Feature called '1300 Airport Road, North Brunswick Township, NJ 08902, USA' at latitude: 404663628, longitude: -744820157
Feature called '' at latitude: 407113723, longitude: -749746483
Feature called '' at latitude: 402133926, longitude: -743613249
Feature called '' at latitude: 400273442, longitude: -741220915
Feature called '' at latitude: 411236786, longitude: -744070769
Feature called '211-225 Plains Road, Augusta, NJ 07822, USA' at latitude: 411633782, longitude: -746784970
Feature called '' at latitude: 415830701, longitude: -742952812
Feature called '165 Pedersen Ridge Road, Milford, PA 18337, USA' at latitude: 413447164, longitude: -748712898
Feature called '100-122 Locktown Road, Frenchtown, NJ 08825, USA' at latitude: 405047245, longitude: -749800722
Feature called '' at latitude: 418858923, longitude: -746156790
Feature called '650-652 Willi Hill Road, Swan Lake, NY 12783, USA' at latitude: 417951888, longitude: -748484944
Feature called '26 East 3rd Street, New Providence, NJ 07974, USA' at latitude: 407033786, longitude: -743977337
Feature called '' at latitude: 417548014, longitude: -740075041
Feature called '' at latitude: 410395868, longitude: -744972325
Feature called '' at latitude: 404615353, longitude: -745129803
Feature called '611 Lawrence Avenue, Westfield, NJ 07090, USA' at latitude: 406589790, longitude: -743560121
Feature called '18 Lannis Avenue, New Windsor, NY 12553, USA' at latitude: 414653148, longitude: -740477477
Feature called '82-104 Amherst Avenue, Colonia, NJ 07067, USA' at latitude: 405957808, longitude: -743255336
Feature called '170 Seven Lakes Drive, Sloatsburg, NY 10974, USA' at latitude: 411733589, longitude: -741648093
Feature called '1270 Lakes Road, Monroe, NY 10950, USA' at latitude: 412676291, longitude: -742606606
Feature called '509-535 Alphano Road, Great Meadows, NJ 07838, USA' at latitude: 409224445, longitude: -748286738
Feature called '652 Garden Street, Elizabeth, NJ 07202, USA' at latitude: 406523420, longitude: -742135517
Feature called '349 Sea Spray Court, Neptune City, NJ 07753, USA' at latitude: 401827388, longitude: -740294537
Feature called '13-17 Stanley Street, West Milford, NJ 07480, USA' at latitude: 410564152, longitude: -743685054
Feature called '47 Industrial Avenue, Teterboro, NJ 07608, USA' at latitude: 408472324, longitude: -740726046
Feature called '5 White Oak Lane, Stony Point, NY 10980, USA' at latitude: 412452168, longitude: -740214052
Feature called 'Berkshire Valley Management Area Trail, Jefferson, NJ, USA' at latitude: 409146138, longitude: -746188906
Feature called '1007 Jersey Avenue, New Brunswick, NJ 08901, USA' at latitude: 404701380, longitude: -744781745
Feature called '6 East Emerald Isle Drive, Lake Hopatcong, NJ 07849, USA' at latitude: 409642566, longitude: -746017679
Feature called '1358-1474 New Jersey 57, Port Murray, NJ 07865, USA' at latitude: 408031728, longitude: -748645385
Feature called '367 Prospect Road, Chester, NY 10918, USA' at latitude: 413700272, longitude: -742135189
Feature called '10 Simon Lake Drive, Atlantic Highlands, NJ 07716, USA' at latitude: 404310607, longitude: -740282632
Feature called '11 Ward Street, Mount Arlington, NJ 07856, USA' at latitude: 409319800, longitude: -746201391
Feature called '300-398 Jefferson Avenue, Elizabeth, NJ 07201, USA' at latitude: 406685311, longitude: -742108603
Feature called '43 Dreher Road, Roscoe, NY 12776, USA' at latitude: 419018117, longitude: -749142781
Feature called 'Swan Street, Pine Island, NY 10969, USA' at latitude: 412856162, longitude: -745148837
Feature called '66 Pleasantview Avenue, Monticello, NY 12701, USA' at latitude: 416560744, longitude: -746721964
Feature called '' at latitude: 405314270, longitude: -749836354
Feature called '' at latitude: 414219548, longitude: -743327440
Feature called '565 Winding Hills Road, Montgomery, NY 12549, USA' at latitude: 415534177, longitude: -742900616
Feature called '231 Rocky Run Road, Glen Gardner, NJ 08826, USA' at latitude: 406898530, longitude: -749127080
Feature called '100 Mount Pleasant Avenue, Newark, NJ 07104, USA' at latitude: 407586880, longitude: -741670168
Feature called '517-521 Huntington Drive, Manchester Township, NJ 08759, USA' at latitude: 400106455, longitude: -742870190
Feature called '' at latitude: 400066188, longitude: -746793294
Feature called '40 Mountain Road, Napanoch, NY 12458, USA' at latitude: 418803880, longitude: -744102673
Feature called '' at latitude: 414204288, longitude: -747895140
Feature called '' at latitude: 414777405, longitude: -740615601
Feature called '48 North Road, Forestburgh, NY 12777, USA' at latitude: 415464475, longitude: -747175374
Feature called '' at latitude: 404062378, longitude: -746376177
Feature called '' at latitude: 405688272, longitude: -749285130
Feature called '' at latitude: 400342070, longitude: -748788996
Feature called '' at latitude: 401809022, longitude: -744157964
Feature called '9 Thompson Avenue, Leonardo, NJ 07737, USA' at latitude: 404226644, longitude: -740517141
Feature called '' at latitude: 410322033, longitude: -747871659
Feature called '' at latitude: 407100674, longitude: -747742727
Feature called '213 Bush Road, Stone Ridge, NY 12484, USA' at latitude: 418811433, longitude: -741718005
Feature called '' at latitude: 415034302, longitude: -743850945
Feature called '' at latitude: 411349992, longitude: -743694161
Feature called '1-17 Bergen Court, New Brunswick, NJ 08901, USA' at latitude: 404839914, longitude: -744759616
Feature called '35 Oakland Valley Road, Cuddebackville, NY 12729, USA' at latitude: 414638017, longitude: -745957854
Feature called '' at latitude: 412127800, longitude: -740173578
Feature called '' at latitude: 401263460, longitude: -747964303
Feature called '' at latitude: 412843391, longitude: -749086026
Feature called '' at latitude: 418512773, longitude: -743067823
Feature called '42-102 Main Street, Belford, NJ 07718, USA' at latitude: 404318328, longitude: -740835638
Feature called '' at latitude: 419020746, longitude: -741172328
Feature called '' at latitude: 404080723, longitude: -746119569
Feature called '' at latitude: 401012643, longitude: -744035134
Feature called '' at latitude: 404306372, longitude: -741079661
Feature called '' at latitude: 403966326, longitude: -748519297
Feature called '' at latitude: 405002031, longitude: -748407866
Feature called '' at latitude: 409532885, longitude: -742200683
Feature called '' at latitude: 416851321, longitude: -742674555
Feature called '3387 Richmond Terrace, Staten Island, NY 10303, USA' at latitude: 406411633, longitude: -741722051
Feature called '261 Van Sickle Road, Goshen, NY 10924, USA' at latitude: 413069058, longitude: -744597778
Feature called '' at latitude: 418465462, longitude: -746859398
Feature called '' at latitude: 411733222, longitude: -744228360
Feature called '3 Hasta Way, Newton, NJ 07860, USA' at latitude: 410248224, longitude: -747127767
-------------- RecordRoute --------------
Visiting point latitude: 412856162, longitude: -745148837
Visiting point latitude: 418465462, longitude: -746859398
Visiting point latitude: 406589790, longitude: -743560121
Visiting point latitude: 416851321, longitude: -742674555
Visiting point latitude: 415301720, longitude: -748416257
Visiting point latitude: 407033786, longitude: -743977337
Visiting point latitude: 401012643, longitude: -744035134
Visiting point latitude: 412144655, longitude: -743949739
Visiting point latitude: 414638017, longitude: -745957854
Visiting point latitude: 400273442, longitude: -741220915
Finished trip with 10 points 
Passed 10 features 
Travelled 850915 meters 
It took 0 seconds 
-------------- RouteChat --------------
Sending First message at latitude: 0, longitude: 0
Sending Second message at latitude: 0, longitude: 1
Sending Third message at latitude: 1, longitude: 0
Sending Fourth message at latitude: 0, longitude: 0
Sending Fifth message at latitude: 1, longitude: 0
Received message First message at latitude: 0, longitude: 0
Received message Third message at latitude: 1, longitude: 0
```




