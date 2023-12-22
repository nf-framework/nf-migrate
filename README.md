# @nfjs/migrate
Database migration tool

**ОГЛАВЛЕНИЕ**
- [Назначение](#Назначение)
  - [Состав инструмента](#Состав-инструмента)
  - [Принцип работы](#Принцип-работы)
  - [Конфигурация инструмента](#Конфигурация-инструмента)
    - [Описание параметров](#Описание-параметров)
## Назначение
Инструмент, позволяющий провести обновление схемы базы данных установленного приложения из исходников,
сохраненных разработчиком в соотвествующие папки каждого модуля(пакета) системы.

## Состав инструмента

### 1. Объекты бд, необходимые для работы самого инструмента
* public.nf_migrations - таблица с именами выполненных уже миграций и временем их применения. 
* public.nf_objects - таблица с именами объектов бд, которые были созданы или изменены этим инструментом и хешем текста файла исходника объекта, 
примененного в предыдущий запуск инструмента. Хеш нужен для быстрой проверки при старте обновления — изменялся ли объект с последнего обновления. Так же здесь хранится и хеш содержимого файла данных таблиц, сохраненных
в исходники
* public.nf_get_objsrc - функция, получающая исходный код указанного объекта в бд. Используется как при сохранении
объекта разработчиков в исходники, так и при сравнении текущего состояния объекта в бд и исходника из репозитория.

### 2. Исходники 
Находятся в папках `./dbsrc` модулей. Для каждой схемы бд создается под-папка `./имясхемы`. 
   
!> Важно, что схема может быть только в одном из пакетов приложения. И в данный момент при добавлении новой схемы бд в приложение нужно
вручную создать в необходимом модуле папку с именем схемы и внутри неё под-папки `dat`, `src`, `mig`.
### 2.1. Объекты бд 
Хранятся в под-папке `./dbsrc/src` и дальше по типам объектов
* функции, представления, триггеры хранятся в виде готового к выполнению sql
* таблицы, последовательности в виде объекта json. Нужно так для библиотеки сравнивания и генерации изменений.
В таблицах поля, ограничения, индексы отсортированы по имени для более наглядной истории в git.
### 2.2. Миграции 
Хранятся в под-папках `./dbsrc/mig/год-месяц`. Разбиваются по месяцам для того чтобы папки не были по истечении
времени перегружены количеством файлов. Миграции используются для случаев, когда нужно провести операции, которые
не может сделать библиотека генерации изменений таблиц, команды изменения данных.
Миграции могут быть разделены на блоки строками вида `--[block]{"event":"run","when":"before"}` в которых описано к 
какому типу событию (`event`) они относятся и когда (`when`) их выполнять.

!> Нельзя управлять в скрипте миграции глобальной транзакцией — весь процесс обновление проходит в одной транзакции.
### 2.3. Данные 
Хранятся в под-папках `./dbsrc/dat/имя_таблицы_данных/`. Это набор данных таблицы, с которым будет синхронизированы данные в таблице на целевой базе обновления.
Используются как правило три файла для каждой таблицы:
* `data.json` подготовленные и выгруженные по `eSch.json` схеме данные таблицы
* `iSch.json` схема импорта данных из data.json в бд. Применяется при обновлении на целевой базе. 
* `eSch.json` схема экспорта из бд, где подготавливаются данные к обновлению, в data.json. Используется разработчиком при подготовке исходников  
> Для `nfc.modulelist` - `iSch.json` и `eSch.json` расположены в модуле инструмента в папке `./data_schemas`.

Выгрузка и загрузка данных происходит с помощью модуля **@nf/ei**, подробное описание в `readme.md` модуля.
Данному инструменту нужны сами данные `data.json` и `iSch.json` по ним готовится массив sql вида `insert on conflict do update` для каждой записи таблицы и зависимых при наличии.
Для этого все схемы `iSch.json` используют **extract** часть типа `json` и **load** типа `execSqlArray`.
## Принцип работы

### Начало
При старте приложения, в зависимости от настройки модуля `@nf/migrate.doMigrate` (или соотвествующего параметра командной строки), запускается проверка необходимости применения изменений
в подключенный провайдер с именем `default` под пользователем, указанном в настройке `@nf/migrate.data_providers.default.checkUserName`. Для этого 
1. Делается проверка наличия объектов инструмента в бд `public.nf_migrations` и `public.nf_objects` и если есть, то достается информация из них.
2. Ищутся в модулях все исходники объектов, данных и миграции. При заполненном `onlySchemas`, оставляются только указанные.
3. Производится сравнение сохраненных хешей объектов из пункта 1, и хешей найденных исходников в пункте 2 для определения какие объекты подлежат обновлению. Определяются необходимые к прогону
миграции — те, которые есть в исходниках и нет в `public.nf_migrations`.
4. Если есть изменения, то переход к подготовке применения

### Подготовка изменений
0. Далее описано для интерактивного режима выполнения обновления `doSilent = false`. Для `doSilent = true` выполняется то же самое,
   только не запрашиваются значения переменных от запустившего процесс обновления, а берутся из соответствующих ключей конфига или переменных командной строки
1. Запрашивается имя и пароль пользователя владельца объектов и режим прогона (значения по-умолчанию настраиваются в конфигурационном файле)
2. Создаются (если первая установка) или модифицируются объекты инструмента
3. Если есть в модулях информация о системных объектах бд (расширениях postgresql), запрашивается имя и пароль супер-пользователя кластера бд
4. Файлы миграций разбиваются на блоки (если они присутствуют в миграции) и проставляется событие из мета-описания блока
5. Для файлов данных запускается преобразование их в упорядоченный по зависимостям внутри схем массив отдельных команд
обновления данных (insert on conflict update)

### Применение изменений (формирования скрипта изменений в зависимости от режима запуска)

1. Выполнение блоков миграции для прогона перед сравнением объектов: `event` = `run`, `when` = `before`. 
Например, переименование колонки, сложные настройки таблиц\индексов не поддерживаемые инструментом сравнения
2. Для файлов исходников, помеченных к применению, запускается генерация блоков скриптов изменений. Блоки:
    * `safedrop` безопасное удаление объекта, которое не повлечет потерю данных. В том числе и временные удаления
    с последующим созданием обновленной версии объекта для решения проблем с зависимостями объектов друг от друга. Примечание для ключей, ограничений, индексов таблиц - полному удалению подлежат только если имена соответствуют 
    рекомендациям наименований, остальные не трогаются, чтобы на конкретном стенде можно было добавить индекс по усмотрению
    компетентного сотрудника и он не удалился при обновлении.
    * `unsafedrop` небезопасное, как удаление колонки, сейчас собираются, но не применяются по-умолчанию. Будет отдельный запрос от пользователя на выполнение этих блоков (не рекомендуется)
    * `main` основные изменения колонок таблиц и полное создание таблиц. Если есть изменение типа колонки, то производится поиск зависимых от этого поля представлений и, если
      эти представления не поменяли свой код, то насильно добавляется их пересоздание
    * `func` создание\пересоздание функций
    * `trig` создание триггеров
    * `view` создание представлений отсортированных в порядке зависимости друг от друга
    * `pkey` создание первичных и уникальных ключей. Отдельно чтобы из следующего блока внешние ключи не нужно было упорядочивать по зависимостям 
    * `end` отложенные на конец обновления объекты, ограничения таблиц, внешние ключи

3. Применение
    1. Выполнение safedrop блока объектов
    2. Запрос на выполнение и выполнение unsafedrop
    3. Выполнение по очереди блоков `main`, `func`, `trig`, `view`, `pkey`
    4. Выполнение изменений в данных таблиц (dat файлы)
    5. Блок `end`
    6. После каждого блока применение блоков из миграций с указанным определенным событием
    7. Выполнение оставшихся блоков из миграций, которые не выполнились по определенным событиям.
4. Отметка всех поучаствовавших объектов, миграций, данных в таблицах инструмента
5. Проверка начальной инициализации приложения (записи в системных справочниках пользователей, ролей) и если всё пусто, то будет запрос на инициализацию с просьбой указать имя первого администратора приложения, его пароль и имя первой роли, которой выдастся минимальный набор привилегий для дальнейшей настройке. Пользователь администратора приложения создается в соотвестии с режим работы провайдера данных `default` - если в режиме `user`, то будет создан соотвествующий пользователь в postgresql, с назначенной ролью `nfusr`, а иначе только запись в `nfc.users`. 
6. Завершение работы в зависимости от выбранного режима (runType).

## Конфигурация инструмента
В основном файле настроек `config.json` приложения добавить ключ `@nf/migrate`
```json
"@nf/migrate": {
        "doMigrate": true,
        "doSilent": false,
        "сheckType": "simple",
        "defaultRunType": "r",
        "defaultDoUnsafeDrop": false,
        "data_providers": {
            "default": {
                "checkUserName": "nfusr",
                "checkUserPassword": "nfusrpass",
                "adminUserName": "nfadm",
                "adminUserPassword": "nfadmpass",
                "superUserName": "postgres",
                "superUserPassword": "postgrespass"
            }
        },
        "defaultDoInit": {
            "need": true,
            "appAdminName": "admin",
            "appAdminPassword": "adminpass"
        }   
    },
```
Также поддерживается режим обновления из командной строки
```bash
node index --domigrate=true --migrate-do-silent=false --migrate-run-type=t
```

### Описание параметров
|Параметр|Тип|Назначение|Параметр командной строки|
|---|---|---|---|
|`doMigrate`|boolean|Запускать процесса обновлениея бд при старте приложения|domigrate|
|`doSilent`|boolean|Проводить процесс обновления только на значениях по-умолчанию, не запрашивая ничего. Применяется для автоматического обновления|migrate-do-silent| 
|`checkType`|string|Способ проверки необходимости выполнения обновления объектов и данных<br>`simple` - проверка по хешам, сохраненным в *public.nf_objects*<br>`force` - все объекты из исходников сравниваются с бд|migrate-check-type|
|`defaultRunType`|string|Режим запуска при `doSilent = true`<br>`r` - выполнение в бд<br>`t` - тестирование в бд<br>`f` - вывод полного скрипта обновления в файл<br>`v` - вывод полного скрипта обновления в консоль|migrate-run-type| 
|`defaultDoUnsafeDrop`|boolean|Выполнять блок `unsafedrop` из сравнения таблиц или нет. Учитывается только при `doSilent = true`|migrate-do-unsafe-drop|
|`data_providers.default`|Object|Указывается провайдер базы данных, в котором нужно применять обновление||
|`.checkUserName`|string|Имя пользользователя под которым проверяется необходимость применения обновления|migrate-check-user-name|
|`.checkUserPassword`|string|Пароль пользользователя под которым проверяется необходимость применения обновления|migrate-check-user-password|
|`.adminUserName`|string|Имя пользользователя под которым проводится обновление|migrate-admin-user-name|
|`.adminUserPassword`|string|Пароль пользользователя под которым проводится обновление|migrate-admin-user-password|
|`onlySchemas`|Array|Ограничивающий перечень схем базы данных для которых выполнять обновление|migrate-only-schemas|
|`grantAllFunction`|string|Имя функции для назначения прав на объекты после обновления. По-умолчанию nfc.f_db8grant_all|migrate-grant-all-function|
|`defaultDoInit`|Object|Настройки проведения инициализации платформенного пользователя-администратора, организации и минимально необходимых прав у администратора для дальнейшей настройки системы||
|`.need`|boolean|Признак необходимости инициализации|migrate-do-init|
|`.appAdminName`|string|Имя пользователя-администратора|migrate-do-init-app-admin-name|
|`.appAdminPassword`|string|Пароль пользователя-администратора|migrate-do-init-app-admin-password|
|`.appAdminRole`|string|Имя первой роли, которой будут даны минимальные права для дальнейшей настройки приложения|migrate-do-init-app-admin-role|

!> При передаче параметров через командную строку параметры типа boolean можно передать в виде строк `y`,`Y`,`true`,`t`.
А массивы строк в виде строки с разделителем `;`

!> Важно. Для обеспечения защиты от случайных обновлений объектов базы данных желательно иметь отличные пароли
проверяющего пользователя `checkUserPassword` на разных базах.