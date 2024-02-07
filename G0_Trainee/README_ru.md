# Решение

Здесь всё уже выполнено до уровня middle, запускай и будет работать.

Подсматривай сюда, если совсем застрянешь в реализации своего решения.

## Объяснение, почему я написал решение именно так
Задавай оставшиеся вопросы в чате, добавлю сюда ответы.

<details>
<summary>Почему выбран pandas как "ETL инструмент"</summary>
<br>

* Это широко распространённый пакет
* Помогает привыкнуть к концепции DataFrame, опыт с которыми потом можно переиспользовать с PySpark
* Решил начать с чего-то, что не требует ещё одного докер-контейнера (в случае с PySpark), чтобы не раздувать эпизод
</details>
<br>

<details>
<summary>Почему какие-то функции вынесены в utils, а какие-то -- нет</summary>
<br>

* В utils функции, которые используются в разных дагах (импортируются несколько раз)
* Также туда вынесены функции, которые в дальнейшем скорее всего будут переиспользоваться
* Кастомные, "одноразовые" функции оставил внутри .py файла с дагом

</details>
<br>

<details>
<summary>Зачем сохранять результаты промежуточных вычислений в файл (csv)</summary>
<br>

* Airflow, даже с TaskFlow, не предполагает прямой передачи данных между тасками (потому что иначе используется XCOM, который задействует бэкенд-базу для обмена мета-данными и не масштабируется под Big Data)
* Чтобы обратиться из одной таски к данным из другой таски, нужно положить результат в какое-либо хранилище (сериализовать и сохранить)
* Этапы разделены на таски, чтобы сохранить масштабирование. Сами процессы тех же Transform и Load могут быть сильно сложнее и дольше, поэтому объединять их с Extract не стоит
* И при разделении видно статус тасок и есть возможность их выполнить заново (очень удобно при отладке и потом при работе), внеся правку в DAG и сделав Clear на Task Instance

</details>
<br>

<details>
<summary>Какие применил Best Practice</summary>
<br>

Python:
* Вынес повторяющиеся операции в функции, а повторяющиеся функции -- в отдельный импортируемый пакет (принцип DRY)
* Type hints для параметров функции и её результата
* Глобальные переменные названы в UPPERCASE, внутренние функции начинаются с "_"
* Указываю кодировку файлов при записи и чтении, а разделитель (sep) выбираю в виде символа, который не может встретиться в данных
* Длинные названия переменных заменяют большинство комментариев, код стремится к "самодокументируемости"
* При обработке ошибок используются по возможности точные Exception'ы

Хранилища:
* Записываю время в UTC
* Добавляю технические поля для дальнейшей работы с данными
* Формирую DDL для таблиц и создаю их отдельным шагом перед вставкой данных
* Использую быструю и надёжную идемпотентную загрузку в staging через truncate - insert
* Использую инкрементальную загрузку в target

Airflow:
* Указал doc-string в начале каждого дага и почти каждой функции
* Логирование на уровнях debug и info
* Использую теги для организации DAG'ов
* Формирую имя промежуточных файлов так, чтобы повторный запуск перезатирал ранее созданный файл для task instance, а новый запуск создавал новый файл
* Даг не запускается повторно, пока не отработает предыдущий (чтобы не испортить staging)
* Для организации тасок используются таск группы, начинающий и завершающий пустые таски
* Обработка разделена на шаги, почти каждый из которых можно безопасно перезапустить (кроме неидемпотентного load, но это дискусионный вопрос)
* Где удобно, использую подходящие для этого операторы вместо хуков и custom кода на python
* Использую Variables, Connections

</details>
<br>

<details>
<summary>Как можно улучшить это решение</summary>
<br>

* Генерировать даги для каждой таблицы отдельно, уменьшить связность
* Импортировать пакеты внутри функций, где они используются
* При заборе данных из источника перечислять колонки через запятую явно, не использовать SELECT * FROM
* Вынести в отдельные функции чтение и запись из промежуточных файликов, чтобы абстрагироваться от CSV формата и иметь возможность заменить в 1 месте
* Обновление лучше бы сделать батчами, а не большим запросом
* Конфигурировать ещё больше параметров, таких как "минимальное и максимальное время для eff_* колонок", ширину окна забора возможно изменившихся данных

</details>