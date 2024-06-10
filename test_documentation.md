# Руководство пользователя
## Введение
Документ содержит описание установки драйвера PostgreSQL ODBC (ODBC) и его возможностей по использованию данных PostgreSQL в сторонних ПО. Под сторонними ПО понимается Oracle, Excel, Access. Описана возможность импорта данных из PostgreSQL в  Excel или Access. Также в документе представлено описание подключения Oracle к PostgreSQL.  
## Использование данных PostgreSQL в стороннем ПО
Штатным средством для объединения разнородных источников данных является драйвер PostgreSQL ODBC.
### Установка драйвера на ОС Windows
Необходимо найти и установить PostgreSQL ODBC driver (psqlODBC). [Ссылка](https://www.postgresql.org/ftp/odbc/) на официальный сайт для скачивания файла.

**Внимание! Важно выбрать правильную разрядность.**  
Если приложение, которому нужен доступ, 32-битное, а драйвер 64-битный, выйдет ошибка:  

    ERROR [IM014] [Microsoft][ODBC Driver Manager] The specified DSN contains an architecture mismatch

Драйвер ODBC лучше всего установить из пакета  .msi.  
Если Windows 64-битная, а драйвер 32-битный, то панель управления нужно вызвать вручную: 

    c:\windows\system32\odbcad32.exe

Во всех остальных случаях интерфейс настройки вызывается штатно через: 

    Control Panel -> Administrative Tools -> Data Sources (ODBC) 
На вкладке “System DSN” необходимо создать новый источник данных, выбрав драйвер “Postgres (unicode)” и прописать учетные данные.
### Установка драйвера на ОС Unix
Для установки драйвера ODBC на ОС Unix необходимо учитывать особенности системы.  
В файл .odbc.ini прописать настройки:  

    [ODBC Data Sources]
    Product = PostgreSQL
    [Product]
    Debug = 1
    CommLog = 1
    ReadOnly = no
    Driver = /usr/pgsql-9.1/lib/psqlodbc.so
    Servername = <PostgreSQL_IP>
    FetchBufferSize = 99
    Username = <user>
    Password = <pass>
    Port = 5432
    Database = <db_name>
    [Default]
    Driver = /usr/lib64/liboplodbcS.so.1

  
Если после установки драйвера он не работает, значит что-то сделано не так. Необходимо проверить версии, разрядность, настройки и нажать кнопку “reset”.
## Создание в Oracle ссылки на PostgreSQL
**Внимание!** Разрядность драйвера должна быть такая же, какая у БД Oracle.

### ОС Windows 
Рассмотрим пример, в котором база данных (не клиент!) установлена под Windows в каталог:

    c:\oracle\product\11.2.0\database

Заходим в каталог:  

    c:\oracle\product\11.2.0\database\hs\admin\

В файле initProduct.ora, где пишем:

    HS_FDS_CONNECT_INFO = Product
    HS_FDS_TRACE_LEVEL = 0
    HS_NLS_NCHAR = UCS2
    HS_LANGUAGE = american_america.we8mswin1252

Product - это имя, созданного ранее источника (может быть любым) здесь и далее используется Product, который нужно создать вручную.  

Последняя строка - не опечатка. Можно еще попробовать так:  

    HS_LANGUAGE = american_america.al32utf8

Но у меня не заработало.  
### ОС Unix
Для ОС Unix скорее всего необходимо будет  прописать строки:

    HS_FDS_CONNECT_INFO = MoodlePostgres
    HS_FDS_SHAREABLE_NAME = /<path_to_postrges>/psqlodbc.so
    HS_FDS_SUPPORT_STATISTICS = FALSE
    HS_KEEP_REMOTE_COLUMN_SIZE = ALL

Заходим в каталог:

    c:\oracle\product\11.2.0\database\NETWORK\ADMIN\

Правим файл: listener.ora  
Необходимо добавить запись в секцию SID_LIST_LISTENER/SID_LIST:  

    SID_LIST_LISTENER =
    (SID_LIST =
    (SID_DESC =
    (SID_NAME = CLRExtProc)
    (ORACLE_HOME = C:\oracle\product\11.2.0\database)
    (PROGRAM = extproc)
    (ENVS = "EXTPROC_DLLS=ONLY:C:\oracle\product\11.2.0\database\bin\oraclr11.dll")
    )
    (SID_DESC=
    (SID_NAME = Product)
    (ORACLE_HOME = C:\oracle\product\11.2.0\database)
    (PROGRAM = dg4odbc)
    )
    )

Добавить описание в файл  tnsnames.ora в этом же каталоге:
 
    Product =
            (DESCRIPTION = 
            (ADDRESS = (PROTOCOL = TCP)(HOST = 127.0.0.1)(PORT = 1521))
            (CONNECT_DATA = 
            (SID = Product))
            (HS = OK)
            )


Сервер Product - подставить свой.  
Порт - стандартный порт Oracle.

Перезапустить “Oracle Database Listener”.  
Выполнить стандартное создание линка:

     create database link Product connect to "Product_scr" identified by "password" using 'Product';
  
Product_scr - пользователь владельца схемы базы Product  
password - пароль базы Product  

Получившуюся ссылку можно использовать следующим образом:

    select* 
    from "Product_rate_plans"@Product;

При написании запроса необходимо имя таблицы обрамлять в кавычки. Остальной синтаксис не отличается от стандартного.  
Для работы с двумя БД используется механизм распределенных транзакций. При изменении данных в рамках транзакции могут возникнуть проблемы. Данная проблема возникает при использовании  нестандартных источников данных. Выполнение операции  SELECT происходит корректно.  
Допустимо программировать процедуры с автономными транзакциями, а также создавать материализованное представление и т.п.
  
## Импорт данных в Excel и  Access
Драйвер ODBC можно использовать с любыми совместимыми приложениями. Драйвер ODBC является продуктом Microsoft, поэтому совместим с большинством его других продуктов. Например, используя данный драйвер,  Microsoft  Access / Excel могут импортировать данные из PostgreSQL.
### Импорт данных в Excel
Для получения данных из PostgreSQL в Excel  необходимо выполнить следующие действия:
1. Через Excel сгенерировать файл динамического запроса к данным .dqy.
2. В файле прописать настройки:

        XLODBC
        1
        DRIVER={PostgreSQL Unicode};DATABASE=Product;SERVER=127.0.0.1;PORT=5432;UID=<user>;PASSWORD=<passwordd>;SSLmode=disable;ReadOnly=0;Protocol=7.4;FakeOidIndex=0;ShowOidColumn=0;RowVersioning=0;ShowSystemTables=0;ConnSettings=;Fetch=100;Socket=4096;UnknownSizes=0;MaxVarcharSize=255;MaxLongVarcharSize=8190;Debug=0;CommLog=0;Optimizer=0;Ksqo=1;UseDeclareFetch=0;TextAsLongVarchar=1;UnknownsAsLongVarchar=0;BoolsAsChar=1;Parse=0;CancelAsFreeStmt=0;ExtraSysTablePrefixes=dd_;LFConversion=1;UpdatableCursors=1;DisallowPremature=0;TrueIsMinus1=0;BI=0;ByteaAsLongVarBinary=0;UseServerSidePrepare=0;LowerCaseIdentifier=0;GssAuthUseGSS=0;XaOpt=1
        select * from Product_rate_plans

SERVER  - указать адрес сервера  
UID - указать пользователя  
PASSWORD - пароль для подключения

Всего должно получиться 4 строки.  Запрос в последней строке.  
**Внимание!** Запрос обязательно должен занимать 1 строку, иначе работать не будет.

3. Сохранить файл.
4. При запуске файла появится диалоговое окно, в котором необходимо согласиться с включением источника данных.
5. Результат запроса из БД представлен в файле Excel.
### Импорт данных в Access
Для получения данных из PostgreSQL в Access необходимо выполнить следующие действия:  
1. В Microsoft Access выбрать в контекстном меню "Источники данных ODBC".
2. Выбрать нужную таблицу или таблицы  для импорта (допускается загрузить сразу все таблицы).
3. Выбранная таблица или таблицы представлены в Microsoft Access.
## Глоссарий
|Наименование|Описание|  
|------------|--------|  
|PostgreSQL ODBC |Решение для подключения приложений, совместимых с ODBC, к базам данных PostgreSQL из Windows, Unix|  
|Microsoft Office Access (Access) |Реляционная система управления базами данных |
|Oracle |Объектно-реляционная система управления базами данных компании Oracle
## Список принятых сокращений
| Сокращение |Описание |  
|------------|----------|  
|БД |База данных|  
|ОС |Операционная система|  
|ПО |Программное обеспечение|  
## Используемая литература
1. http://stackoverflow.com/questions/6796252/setting-up-postgresql-odbc-on-windows
2. https://dbaspot.wordpress.com/2013/05/29/how-to-access-postgresql-from-oracle-database/
3. https://community.oracle.com/thread/2549585
4. http://www-01.ibm.com/support/docview.wss?uid=swg21651061


