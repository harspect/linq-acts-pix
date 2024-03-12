
# 1BIT-NN PIX-REFramework_NoQueue
## Описание проекта
**Robotic Enterprise Framework** — это шаблон проекта, основанный на State Machines, создан с учетом всех передовых методов ведения журналов, обработки исключений, инициализации приложений и т. д. и готов к решению сложных бизнес-сценариев.

Логика шаблона основана на **REFramework UiPath** и имеет схожую архитектуру.

Версия *"_NoQueue"* предназначена для проектов, архитектура которых не предполагает взаимодействие с очередями PIX Master и не разбивается на **Dispatcher\Performer**, транзакции создаются и обрабатываются за один запуск.

Шаблон разработан совместными усилиями команд **ООО "Первый бит"** и **ПАО «ГМК "Норильский Никель"»**
___

## Содержание:
1. [Функциональные преимущества](#functional-benefits)
2. [Файловая структура проекта](#project-file-structure)
3. [Параметры](#arguments)
4. [Main.pix](#main-pix)
5. [STATE 1: Инициализация](#state-1)
6. [STATE 2: Общие действия для всех транзакций](#state-2)
7. [STATE 3: Получение данных транзакции](#state-3)
8. [STATE 4: Обработка/создание транзакции](#state-4)
9. [STATE 5: Завершение](#state-5)
10. [InitAllSettings.pix](#init-all-settings-pix) 
11. [Чтение конфигурационного файла](#read-config)
12. [KillProcesses.pix](#kill-processes-pix)
13. [Notification.pix](#notification-pix)
14. [TakeScreenshot.pix](#take-screenshot-pix)
15. [CreateTransactions.pix](#create-transactions-pix)
16. [GetTransaction.pix](#get-transaction-pix)
17. [PreProcess.pix](#pre-process-pix)
18. [Process.pix](#process-pix)
19. [Finalize.pix](#finalize-pix)
23. [Памятка по эксплуатации](#exploitation)


### <a id="functional-benefits">Функциональные преимущества</a>
* Реализовано без использования **"GoTo"**.
* Возможность переключения среды (**тестирование, продуктив**) для запуска робота без изменений в проекте.
* Использование специализированных словарей для хранения данных.
* Возможность указания данных **PIX Master (Text, Secure Data, Authorization), Windows Credentials** и **String** значений в конфигурационном файле с дальнейшим распределением внутри проекта в соответствующие словари.
* Реализованы дополнительные функциональные скрипты (**Отправка сообщений, Скриншот и т.п.**).
* Возможность ограничения количества транзакций для обработки за один запуск.
* Реализованы попытки выполнения транзакций.
* Автоматическое формирование таблицы со статусами обработки транзакций.
___

### <a id="project-file-structure">Файловая структура проекта</a>
**REFramework_Queue**
* **Data**
  * Config.xlsx
* **FrameworkScripts**
  * InitAllSettings.pix
  * KillProcesses.pix
  * Notification.pix
  * TakeScreenshot.pix
* **TransactionScripts**
  * CreateTransactions.pix
  * Finalize.pix
  * GetTransaction.pix
  * PreProcess.pix
  * Process.pix
* **Main.pix**
___

### <a id="variables">Переменные</a>

|Название|Тип данных|Описание|
|--|--|--|
|exc_main|System.Exception/BusinessRuleException|Хранение ошибки, возникающей во время работы робота|
|dict_config|Dictionary<String, Object>|Словарь для данных конфигурационного файла и других значений|
|dict_secureData|Dictionary<String, SecureString>|Словарь для хранения данных учетных записей|
|dict_paths|Dictionary<String, String>|Словарь для хранения путей к проектным\созданным папкам|
|int_transCount|Integer|Счетчик количества транзакций, которые были обработаны за один запуск|
|str_transactionID|String|ID транзакции|
|row_transactionItem|DataRow|Транзакция (строка таблицы)|
|dt_transactions|DataTable|Таблица транзакций|
|int_state|Integer|Состояние, с которого начинается цикл|
|int_transactionRetryCounter|Integer|Счетчик повторных попыток обработки транзакции|
|bool_exit|Boolean|Флаг выхода из цикла|

### <a id="arguments">Параметры</a>

|Название|Тип данных|Описание|
|--|--|--|
|int_transNumber|Integer|Количество транзакций, которое должно быть обработано за один запуск. 0 - без ограничений|
|bool_isProd|Boolean|true - запуск в продуктиве, false - запуск в предпродуктиве\тесте|
|bool_isPredProd|Boolean|true - запуск в предпродуктиве, false - запуск в продуктиве\тесте|
|int_maxTransactionRetry|Integer|Макс. кол-во попыток обработки транзакции|

---

## <a id="main-pix">Main.pix</a>

 **Схема работы State Machine**
 
![StateMachine](https://github.com/First-BIT-v2-0/Share/blob/main/Framework_NoQueue.png?raw=true)


### <a id="state-1">STATE 1: Инициализация</a>

#### Описание
Блок предназначен для первичной настройки окружения и считывания конфигурационных данных.
В процессе подготовки робот:
1. Записывает лог о начале работы.
2. Закрывает приложения, которые могут помешать работе текущего процесса.
3. Считывает конфигурационный файл.
4. Инициализирует проектные папки.

#### Обработка ошибок 
Ошибка записывается **out** параметром блока **Catch** в переменную **exc_main** для последующей обработки.
Источник ошибки перезаписывается отдельным шагом блока **Catch** для корректного вывода скрипта\шага ошибки.

#### Переходы
* Есть ошибки - переход в **STATE 5: Завершение**
* Нет ошибок  - переход в **STATE 2: Общие действия для всех транзакций**

---

### <a id="state-2">STATE 2: Общие действия для всех транзакций </a>

#### Описание
Блок предназначен для наполнение пула транзакций для дальнейшей обработки и выполнения общих действий.
К общим действиям относятся - запуск приложений, авторизация на сайте и т.п.  

**При отсутствии данных для обработки - вызывается бизнес исключение "Отсутствуют новые транзакции для обработки"**

Количество запусков:
> При отсутствии ошибок во время работы робота, **STATE 2** вызывается один раз после инициализации первичных настроек.

>При возникновении ошибок, в зависимости от настроек проекта, **STATE 2** может вызываться несколько раз:
>1. При повторной попытке обработать транзакцию
>2. При изменении логики обработки ошибок транзакций

#### Логика работы блока:
1. Если нет ошибок или нет id транзакции (запуск после инициализации)
    * Заполнить таблицу транзакций для обработки, добавить столбец **"Статус"**.
    * Если в очереди нет транзакций - Вызвать бизнес исключение "Отсутствуют новые транзакции для обработки".
2. Если есть транзакция на обработке и ошибка (перезапуск приложений для повторной попытки обработки транзакции)  - нет действий.
3. Выполнить общие действия - **PreProcess**.

#### Обработка ошибок 
Ошибка записывается **out** параметром блока **Catch** в переменную **exc_main** для последующей обработки.
Источник ошибки перезаписывается отдельным шагом блока **Catch** для корректного вывода скрипта\шага ошибки.

#### Переходы
* Если есть ошибка и ID транзакции **не null** (перезапуск) - переход в **STATE 4: Обработка/создание транзакции**
* Если есть ошибка и ID транзакции **null** - переход в **STATE 5: Завершение**
* Нет ошибок  - переход в **STATE 3: Получение данных транзакции**

---

### <a id="state-3">STATE 3: Получение данных транзакции</a>

#### Описание
Блок предназначен для извлечения следующей транзакции для обработки.
По итогу работу извлекается значение и ID транзакции.

> Если получен сигнал остановки из PIX Master - ID элемента очереди - **null**

#### Обработка ошибок 
Ошибка записывается **out** параметром блока **Catch** в переменную **exc_main** для последующей обработки.
Источник ошибки перезаписывается отдельным шагом блока **Catch** для корректного вывода скрипта\шага ошибки.
**str_transactionID** устанавливается в **null**.

#### Переходы
* Если есть ID транзакции - переход в **STATE 4: Обработка/создание транзакции**
* Если нет ID транзакции - переход в **STATE 5: Завершение**

---

### <a id="state-4">STATE 4: Обработка/создание транзакции</a>

#### Описание
В данной блоке описывается основная бизнес логика процесса обработки транзакции.
В начале данного блока значение переменной **exc_main** сбрасывается.
В случае успешной работы скрипта **Process.pix**  -  транзакции устанавливается статус **Обработано**.
Блок логируется с указанием ID текущей транзакции.

#### Обработка ошибок 
Ошибка записывается **out** параметром блока **Catch** в переменную **exc_main** для последующей обработки.
Источник ошибки перезаписывается отдельным шагом блока **Catch** для корректного вывода скрипта\шага ошибки.
Если превышено количество перезапусков транзакции - транзакции проставляется статус с соответствующим типом ошибки (**Business\System**).

#### Переходы
* **Если ошибка при обработке транзакции**:
  * Если количество перезапусков превышено - завершение приложений, выполнение PreProcess, переход в **STATE 3: Получение данных транзакции**.
  * Если количество перезапусков не превышено - закрыть приложения, переход в **STATE 2: Общие действия для всех транзакций** для повторной обработки.
* **Если нет ошибки при обработке транзакции**:
  * Если превышено максимальное количество транзакций для обработки - переход в  **STATE 5: Завершение**.
  * Если не превышено максимальное количество транзакций для обработки - переход в **STATE 3: Получение данных транзакции**.

---

### <a id="state-5">STATE 5: Завершение</a>

#### Описание
Блок предназначен для завершения работы робота, закрытия приложений, удаления временных файлов, отправки отчетов и обработки критической ошибки при ее наличии.

По умолчанию в данном блоке добавлено условие для действий, которые необходимо выполнить если во время запуска не возникло никаких ошибок - вызов скрипта **Finalize.pix**.

#### Обработка ошибок 
Ошибка записывается **out** параметром блока **Catch** в переменную **exc_main** для последующей обработки.
Источник ошибки перезаписывается отдельным шагом блока **Catch** для корректного вывода скрипта\шага ошибки.

При наличии переданной ошибки из других состояний, в блоке **Finally** происходит вызов исключения с соответствующим содержанием и типом для отображения ошибки в PIX Master.

>Таким образом, если критическая ошибка возникнет на любом этапе работы проекта, блок **STATE 5: Завершение** выполнит финальные действия, закроет все приложения и вызовет ошибку.

#### Переходы
Блок является последним этапом работы робота и не имеет переходов в другие состояния.

---

## <a id="init-all-settings-pix">InitAllSettings.pix</a>

#### Описание:
Скрипт используется для создания проектных папок, чтения конфигурационного файла и прочих настроек.
#### Создание проектных папок
**По умолчанию, скрипт создает следующие папки:**

|Название папки|Путь по умолчанию|Переменная для хранения пути|Назначение|Поведение|
|--|--|--|--|--|
|Templates|.\Templates|dict_paths["TemplatesFolder"]|Папка с шаблонами - входные excel-файлы, sap-скрипты, vba-макросы и т.п.|Создается при отсутствии|
|Results|.\Results|dict_paths["ResultsFolder"]|Папка для результатов работы робота - отчет по транзакциям|Создается при отсутствии|
|ErrorScreenshots|.\ErrorScreenshots|dict_paths["ErrorScrFolder"]|Папка для скриншотов ошибок|Создается при отсутствии|
|TEMP|.\TEMP|dict_paths["TEMPFolder"]|Папка для временных файлов|Пересоздается|

#### <a id="read-config">Чтение конфигурационного файла</a>
Файл Config.xlsx используется для хранения конфигурационных данных робота.

**Скрипт позволяет указывать в файле данные различного типа, которые будут храниться в соответствующем словаре:**

|Тип в файле|Тип данных|Из PIX Master|
|--|--|:--:|
|AuthCredentials|AuthCredentials | ✓ |
|SecureData|SecureData | ✓ |
|Text|Text, Int, Bool | ✓ |
|WinCred|Windows Credentials | ✗ |
||Text | ✗ |

> При пустом значении - используются данные напрямую из Config.xlsx

**В зависимости от указанного типа, данные записываются в словари **dict_config** и **dict_secureData**:**

|Тип данных|Словарь|Ключ|
|--|--|--|
|AuthCredentials|dict_config|"Имя_login"|
|AuthCredentials|dict_secureData|"Имя_passwd"|
|SecureData|dict_secureData|"Имя"|
|WinCred|dict_config|"Имя_login"|
|WinCred|dict_secureData|"Имя_passwd"|
|Text|dict_config|"Имя"|

**Лист Settings**
Предназначен для определения общих конфигурационных данных под все типы информационных сред разрабатываемого проекта.

**Лист Data_Separation** 
Предназначен для разделения конфигурационных данных под каждую информационную среду (PROD, PREDPROD, TEST).


#### Пример чтение конфигурационного файла
**Лист Settings**

|Имя|Значение|Тип|
|--|--|--|
|MailSubject|Robot-XXXX-Test||
|MailCred|MyEmailCred|WinCred|

**Лист Data_Separation**

|Имя|Значение_Prod|Значение_PredProd|Значение_Test|Тип|
|--|--|--|--|--|
|Message|Main mail message|Test mail message|Hello world||
|Recipients|Users|Users|Admin|Text|

**Результат работы скрипта:**

---
**Значения не зависимые от параметров запуска bool_isProd,bool_isPredProd**

|Переменная|Значение|
|--|--|
|dict_config["MailSubject"]|"Robot-XXXX-Test"|
|dict_config["MailCred_login"]|Логин из WindowsCredentials с именем **MyEmailCred**|
|dict_secureData["MailCred_passwd"]|Пароль из WindowsCredentials с именем **MyEmailCred**|

---

**bool_isProd - true, bool_isPredProd - true/false**

|Переменная|Значение|
|--|--|
|dict_config["Message"]|"Main mail message"|
|dict_config["Recipients"]|Значение данных из PIX Master с именем **Users**|
---

**bool_isProd - false, bool_isPredProd - true**

|Переменная|Значение|
|--|--|
|dict_config["Message"]|"Test mail message"|
|dict_config["Recipients"]|Значение данных из PIX Master с именем **Users**|

---

**bool_isPredProd - false, bool_isProd - false**

|Переменная|Значение|
|--|--|
|dict_config["Message"]|"Hello world"|
|dict_config["Recipients"]|Значение данных из PIX Master с именем **Admin**|
---

#### Обработка ошибок
Ошибка передается на уровень выше.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|str_configFile|String|Путь к конфигурационному файлу|
|dict_config|Dictionary<String, Object>|Конфигурационный словарь для хранения данных|
|bool_isProd|Boolean|Чтение данных для среды Prod|
|bool_isPredProd|Boolean|Чтение данных для среду PredProd|
|dict_secureData|Dictionary<String, SecureString>|Словарь для хранения паролей|
|dict_paths|Dictionary<String, String>|Словарь для хранения путей|

> При указании параметров bool_isProd - false и bool_isPredProd - false, читаются данные столбца Значение_Test
---

## <a id="kill-processes-pix">KillProcesses.pix</a>

#### Описание:
Скрипт "убивает" в активной пользовательской сессии запущенные процессы из списка.
Список названий процессов передается с помощью параметра  скрипта - **list_processesToKill**.

>Если передать пустой список - завершатся все процессы, определенные в скрипте.

Названия процессов некоторых приложений:

|Процесс|Приложение|
|--|--|
|excel|Microsoft Excel|
|winword|Microsoft Word|
|outlook|Microsoft Outlook|
|iexplore|Internet Explorer|
|chrome|Google Chrome|

#### Обработка ошибок
Ошибка передается на уровень выше.
Скрипт не выдает ошибку при попытке закрыть несуществующий процесс.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|list_processesToKill|List<String>|Список названий процессов для завершения|

---

## <a id="notification-pix">Notification.pix</a>

#### Описание:
Скрипт для отправки уведомлений. 

#### Типы уведомлений
В скрипт добавлены три типа уведомлений, которые регулируются параметром **int_notifyType**. Каждый тип уведомлений отправляет сообщение определенной группе пользователей.
Разработчик по своему усмотрению может добавлять дополнительные типы уведомлений, связывая с активностью **Условный оператор** новые контейнеры.

**Настройки по умолчанию**

|Значение int_notifyType|Тип уведомления|Получатели|
|--|--|--|
|1|Успешное завершение работы|Пользователи|
|2|Бизнес-исключение|Пользователи|
|3|Системное исключение|Пользователи и администраторы|

#### Использование dict_config
В аргументе **dict_config** можно передавать любую предварительно добавленную в словарь информацию.
Например, текст ошибки или уведомления, адресатов рассылки, тему письма и т.д. 

**Настройки по умолчанию**

|Значение|Описание|
|--|--|
|dict_config["MailServer"]|Сервер SMTP|
|dict_config["MailCred_login"]|Адрес электронной почты для отправки сообщений|
|dict_config["MailRecip"]|Электронные адреса пользователей|
|dict_config["MailAdmin"]|Электронные адреса администраторов|
|dict_config["MailSubject"]|Тема письма|
|dict_config["MessageBody"]|Тело письма|

#### Вложения для уведомления
Вложения для отправки передаются с помощью параметра **list_attaches**, в котором необходимо указать пути к файлам/папкам для отправки.

Дополнительно, к отправке сообщения можно добавить скриншот экрана, установив значение **bool_takeScreenshot** - true.
#### Обработка ошибок
Ошибка передается на уровень выше.
Если указан несуществующий тип уведомления скрипт выдает ошибку.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|int_notifyType|Integer|Тип уведомления|
|dict_config|Dictionary<String, Object>|Словарь конфигурации|
|list_attaches|List<String>|Коллекция путей к файлам вложений|
|sec_mailPassword|SecureString|Пароль от электронной почты|
|bool_takeScreenshot|Boolean|Необходимо взять скриншот|

---

## <a id="take-screenshot-pix">TakeScreenshot.pix</a>

#### Описание:
Сохранение скриншота по заданному шаблону.
**Шаблон по умолчанию : Тип_Дата_Время**

#### Обработка ошибок
Ошибка передается на уровень выше.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|dict_config|Dictionary<String, Object>|Словарь конфигурации|
|str_screenPath|String|Путь сохранения скриншота|
|str_screenType|String|Тип скриншота|
---

## <a id="create-transactions-pix">CreateTransactions.pix</a>

#### Описание:
Скрипт используется для создания транзакций.
По умолчанию создается таблица **dt_transactions**.

#### Обработка ошибок
Ошибка передается в на уровень выше,  пользователи получают уведомление.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|dict_config|Dictionary<String, Object>|Словарь конфигурации|
|dict_secureData|Dictionary<String, SecureString>|Словарь с паролями|
|dict_paths|Dictionary<String, String>|Словарь с путями|
|out_dt_transactions|DataTable|Заполненная таблица транзакций|

---

## <a id="get-transaction-pix">GetTransaction.pix</a>

#### Описание:
Скрипт для получения данных транзакции из таблицы **dt_transactions**. Выполняется поиск первой транзакции, которая имеет пустой статус.

Идентификатор транзакции - порядковый номер найденной строки (начиная с 1..).

Найденной транзакции устанавливается статус **"В обработке"**, если транзакцию не удалось найти - передается идентификатор транзакции - **null**.

#### Обработка ошибок
Ошибка передается на уровень выше.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|row_transactionItem|Row|Строка из таблицы транзакций|
|str_transactionID|String|Идентификатор элемента очереди|
|int_transCount|Integer|Количество обработанных транзакций|
|dt_transactions|DataTable|Таблица с транзакциями|

---

## <a id="pre-process-pix">PreProcess.pix</a>

#### Описание:
Скрипт предназначен для выполнения подготовительных общих действий, которые необходимо сделать перед стартом скриптов **CreateTransactions.pix**, **Process.pix**. 
В **PreProcess.pix** можно выполнять вход во все системы, участвующие в процессе, а также выполнять, например, считывание шаблонных файлов, определяющих работу бизнес-процесса. 

Инициализацию различных систем рекомендуется выполнять в активности **Try\Fix**.

#### Обработка ошибок
Ошибка передается на уровень выше, пользователи получают уведомление.

#### Параметры
|Имя переменной|Тип данных|Описание|
|--|--|--|
|dict_config|Dictionary<String, Object>|Конфигурационный словарь |
|dict_secureData|Dictionary<String, SecureString>|Словарь с паролями|

---

## <a id="process-pix">Process.pix</a>

#### Описание:
Скрипт для выполнения обработки одной транзакций.
В данный скрипт необходимо добавить все действия, по обработке транзакции.

#### Обработка ошибок
Ошибка передается на уровень выше.
Если количество попыток выполнения скрипта превышает максимально допустимое - пользователи получают уведомление.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|dict_paths|Dictionary<String, String>|Словарь с путями к файлам|
|dict_config|Dictionary<String, Object>|Конфигурационный словарь для хранения данных|
|dict_secureData|Dictionary<String, SecureString>|Словарь с паролями|
|row_transactionItem|DataRow|Строка из таблицы транзакций|
|str_transactionID|String|Идентификатор транзакции|
|int_maxTransactionRetry|Integer|Максимально допустимое количество попыток выполнения транзакции|
|int_transactionRetryCounter|Integer|Количество попыток выполнения транзакции|

---

## <a id="finalize-pix">Finalize.pix</a>

#### Описание:
Скрипт для выполнения дополнительных завершающих действий после обработки транзакций (отправка отчета о работе робота, завершающего письма и т.д.) перед окончательной остановкой робота.

По умолчанию скрипт вызывается из **Main.pix** только при успешной обработки всех транзакций.

#### Обработка ошибок
Ошибка передается на уровень выше, пользователи получают уведомление.

#### Параметры

|Имя переменной|Тип данных|Описание|
|--|--|--|
|dict_paths|Dictionary<String, String>|Словарь с путями к файлам|
|dict_config|Dictionary<String, Object>|Конфигурационный словарь для хранения данных|
|dict_secureData|Dictionary<String, SecureString>|Словарь с паролями|
---

## <a id="exploitation">Памятка по эксплуатации</a>

#### Логика получения и обработки транзакций выполняется за один запуск робота
В случаях, когда разделение робота на **Dispatcher\Performer** не является целесообразным решением, рекомендуется использовать данную версию шаблона.

#### Статус транзакций
По умолчанию скрипт добавляет в таблицу транзакций колонку **"Статус"** и заполняет после обработки каждой транзакции. 
Данный отчет сохраняется в папку **"Results"** для последующего анализа\отправки пользователям.

#### Профили запуска 
В случаях, когда данные для запуска в продуктивной и тестовой среде отличаются (например при разработке необходимо использовать тестовый SAP, отдельную почту и т.п.) рекомендуется указывать данные в **Config.xlsx** в специальных столбцах.

При запуске робота, взависимости от значений параметров запуска, в переменной **dict_config** будут храниться значения из соответствующих столбцов листа **Data_Separation**.

|Параметры запуска|Столбец для считывания|
|--|--|
|bool_isProd - true, bool_isPredProd - true/false|Значение_Prod|
|bool_isProd - false, bool_isPredProd - true|Значение_PredProd|
|bool_isProd - false, bool_isPredProd - false|Значение_Test|

#### Готовый скрипт для отправки сообщений Notification.pix
Для отправки сообщений различного вида рекомендуется использовать скрипт **Notification.pix**, по необходимости можно изменить логику, добавить новый тип уведомлений.

---