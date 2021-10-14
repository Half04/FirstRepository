## Постановка задачи

В компании www.mycompany.com, которая находится в городе Гадюкино, необходимо разработать приложение для отображения расписания кинотеатров этого города. В компании имеются справочники кинофильмов и кинотеатров. Необходимо разработать структуру хранения расписаний с привязкой к справочникам, страницу добавления расписания и страницу отображения расписания города. Нужно уделить внимание валидации вводимых данных.


## Комментарии

### Общие замечания

1. Вообще не реализована часть задания, посвящённая пользовательскому интерфейсу.

2. Отсутсвует логирование и обработка ошибок, не возвращаются коды ошибок.


### Нерациональное использование памяти

1. В методе расширения NullOrEmpty для IEnumerable<> класса CommonExtensions каждый раз создаётся новый Enumerator - это довольно расточительно с точки зрения памяти. Можно переписать этот метод для ICollection<>, имеющий свойство Count.

2. В методах репозиториев GetAll и GetByFilter результат запроса к базе сначала приводится к List<>, а затем к массиву. Происходит лишнее выделение памяти. Имеет смысл сразу приводить результат к массиву. По той же причине в начале методов результат запроса можно инициализировать пустым массивом.

3. В методах контроллеров происходит один асинхронный вызов метода репозитория. В данном случае использование конструкции async/await является избыточным: с точки зрения памяти создаётся ещё один конечный автомат, который по сути не имеет смысла. Нужно просто возвращать Task.


### Дублирование кода, рефакторинг

1. Для репозиториев имеет смысл выделить общую часть и создать базовый generic-класс. Это сократит дублирование кода, а следовательно количество ошибок, связанных с Copy-Paste.

2. Аналогичные изменения можно проделать для контроллеров.

3. В CinemaController метод GetAll(int id) стоит переименовать в Get(int id).

4. Класс Startup в проекте Cinema использует файл appsettings.json, который находится в проекте Cinema.WebHost. Класс Startup в проекте Cinema.WebHost вообще нигде не используется.


### Маршруты

1. Маршруты не совсем соответсвуют REST. Например, для фильмов /movie/getall - получение всех фильмов. Более корректно было бы возвращать все фильмы по адресу /movie, а саму операцию определять методом HTTP (в данном случае GET). Аналогично для остальных: /movie/{id}/get -> /movie/{id} GET, /movie/add -> /movie POST, /movie/remove -> /movie DELETE.

2. Для классов Контроллеров можно применить атрибут RouteAttribute для выделения общего префикса у маршрутов. Например, для класса MovieController: Route("movie") или Route("[controller]") - если имя Контроллера совпадает с префиксом.

3. В классе Startup в проекте Cinema вызывается метод UseMvc, который принимает делегат для формирования маршрутов - но маршруты формируются при помощи атрибутов, поэтому вызов метода не имеет смысла.


### Репозитории

1. В репозиториях отсутсвует какая-либо обработка ошибок - все блоки catch пустые. Получается, что в случае ошибок репозиторий замалчивает, что что-то пошло не так и продолжает выполнение. Можно либо логировать, либо пробрасывать исключение наверх, если репозиторий не знает как его обрабатывать.

2. Все репозитории имеют зависимость от настроек (конфигурации) приложения, из которых затем вытаскивается строка подключения к БД. Следует оставить в качестве зависимости только строку подключения, а не всю конфигурацию.


### Валидация

1. Атрибут ValidateModelAttribute (фильтр действий) имеет смысл применять не к Контроллерам, а только к методам, которые принимают на вход Объект запроса - у всех контроллеров это метод Add.

2. В классе MovieShow для значений идентификаторов Кинотеатра (CinemaId) и Фильма (MovieId) определена область допустимых значений от 1 до int.MaxValue. В классах Cinema и Movie для аналогичных идетификаторов таких проверок нет.


### Формирование SQL-запросов

1. Текст запросов зашит в код в качестве аргументов методов - это не очень читаемо и такой код не очень удобно сопровождать.\
\
Как минимум стоит вынести текст запросов в переменные, а при вызове методов уже пользоваться переменными.

Опционально:

2. В целом возможен подход с реализацией диниамической генерации sql-запроса:\ Что выбираем, Откуда выбираем, Как фильтруем.\
Общую задачу формирования запросов можно назначить отдельному объекту - Построителю запросов.\
Для фильтров можно реализовать Преобразователь, которые на вход принимает Фильтр, а на выходе возвращает строку с выражением "where ...".
В самом простом варианте Фильтр может хранить корневое условие, которое хранит Имя поля (колонки), Значение, Знак (Операцию сравнения), а также ссылку на следущее условие с операцией связывания (And, Or).


### Фильтры

1. Во всех фильтрах есть свойство EntityId. Но это свойство использует только EntityFilter - нарушение принципа Interface Segregation.

2. В целом подход с фильтрами выглядит не очень гибко: мы принимаем фильтр-интерфейс и явно приводим его к конкретному классу. А потом вытаскиваем из него уже конкретные данные.
В одном из методов используются условия вида: IFilter is ConcreteFilter. В зависимости от типа фильтра формирутеся тело запроса в БД. Это также не очень хорошо с точки зрения читаемости и гибкости. Если у нас будет много фильтров, то мы получим огромное количество конструкций if.\
\
Один из подходов решения обозначенных проблем - использование нескольких методов получения: Get - получение сущности по идентификатору, GetByDate - получение сущностей за определённую дату и т.д. Но такой подход допустим, если у нас немного Фильтров или вариаций условий.\
\
Ещё один способ: делегирование формирования части запроса, связанной с фильтрацией (условиями), другой сущности. Если использовать Преобразователь фильтров, описанный ранее, можно оставить один метод GetByFilter, принимать на вход IFilter и динамически формировать тело запроса.

3. В DateFilter отсутсвует проверка заполнения даты.


### База данных, Структура хранения

<b>Таблица MovieShow</b>

1. Сеанс представляет собой уникальную связку Кинотеатр + Фильм + Дата + Время.\
\
1.1 Следует отдельно хранить Дату и Время Сеанса. Это позволит запрашивать время Сеансов фильма на определённый день в конкретном кинотеатре.\
1.2 Следует добавить уникальный констрейнт (ограничение) на связку Кинотеатр + Фильм + Дата + Время - для предотвращения дублирования Сеансов.\
1.3 Для связки (MovieId, CinemaId, Date) можно добавить индекс для возможного ускорения запросов.

2. Для колонок MovieId и CinemaId следует добавить индексы - в целом считается хорошей практикой добавлять индексы для внешних ключей. При операциях соединения таблиц (Join) и точечных запросах возможен выигрыш по времени.

3. Также стоит добавить индекс для Даты, так как в репозитории есть запрос с фильтрацией Сенасов только по дате.

Опционально:

4. Сеансы, отличающиеся только временем, могут накладываться один на другой - т.е. один сеанс ещё не закончился, но уже начался второй. Эта ситуация по-хорошему тоже должна проверяться. Можно дополнительно хранить длительность Сеанса.


<b>Таблица Cinema</b>

1. В модели Кинотеатра задана проверка на длину наименования от 5 до 500. Следует добавить соотвествующую проверку на минимальную длину колонки Name.

<b>Таблица Movie</b>

1. В модели Фильма задана проверка на длину наименования от 5 до 500. Следует добавить соотвествующую проверку на минимальную длину колонки Name.

