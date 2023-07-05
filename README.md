# Alfa
# Задача №1
Требуется создать систему которая будет показывать генеалогическое древо, как бы выглядела структура базы 
данных для хранения данных, из каких таблиц и с какими атрибутами она состояла

## Решение
При создании системы для отображения генеалогического древа, структура базы данных может выглядеть следующим образом:

Таблица People:
* ID (Primary Key) - уникальный идентификатор человека.
* Name - имя человека
* Gender - пол человека
* Birth Date - дата рождения человека.
* Death Date - дата смерти человека
  
Таблица Relationships:
* ID (Primary Key) - уникальный идентификатор связи.
* Person1 ID - ссылка на таблицу "People" (ID первого человека в связи).
* Person2 ID - ссылка на таблицу "People" (ID второго человека в связи).
* Relationship Type - тип родственной связи.

```
CREATE TABLE People (
    ID INTEGER PRIMARY KEY,
    Name TEXT,
    Gender TEXT,
    BirthDate TEXT,
    DeathDate TEXT
);

CREATE TABLE Relationships (
    ID INTEGER PRIMARY KEY,
    Person1ID INTEGER,
    Person2ID INTEGER,
    RelationshipType TEXT,
    FOREIGN KEY (Person1ID) REFERENCES People (ID),
    FOREIGN KEY (Person2ID) REFERENCES People (ID)
);
```
Main.py
```bash
import sqlite3
import pandas as pd
from random import choice, randint, seed
from datetime import date, timedelta

seed(42)

def generate_random_name():
    first_names = ['John', 'Jane', 'Robert', 'Emily', 'Michael', 'Emma', 'William', 'Olivia', 'James', 'Ava']
    last_names = ['Smith', 'Johnson', 'Brown', 'Miller', 'Davis', 'Garcia', 'Wilson', 'Martinez', 'Lee', 'Gonzalez']
    return f'{choice(first_names)} {choice(last_names)}'

def generate_random_birth_date():
    start_date = date(1940, 1, 1)
    end_date = date(2005, 1, 1)
    random_date = start_date + timedelta(days=randint(0, (end_date - start_date).days))
    return random_date.strftime('%Y-%m-%d')

def generate_random_death_date(birth_date):
    birth_date = date.fromisoformat(birth_date)
    start_date = birth_date + timedelta(days=randint(1, 365 * 90))  # Жизнь от 1 дня до 90 лет
    end_date = date(2022, 1, 1)  # До 2022 года
    random_date = min(start_date, end_date)
    return random_date.strftime('%Y-%m-%d')

# Создаем базу данных и таблицы
conn = sqlite3.connect('genealogy_tree.db')
cursor = conn.cursor()

cursor.execute('''
CREATE TABLE People (
    ID INTEGER PRIMARY KEY,
    Name TEXT,
    Gender TEXT,
    BirthDate TEXT,
    DeathDate TEXT
)
''')

cursor.execute('''
CREATE TABLE Relationships (
    ID INTEGER PRIMARY KEY,
    Person1ID INTEGER,
    Person2ID INTEGER,
    RelationshipType TEXT,
    FOREIGN KEY (Person1ID) REFERENCES People (ID),
    FOREIGN KEY (Person2ID) REFERENCES People (ID)
)
''')

for _ in range(50):
    name = generate_random_name()
    gender = 'Male' if choice([True, False]) else 'Female'
    birth_date = generate_random_birth_date()
    death_date = generate_random_death_date(birth_date)

    cursor.execute('INSERT INTO People (Name, Gender, BirthDate, DeathDate) VALUES (?, ?, ?, ?)',
                   (name, gender, birth_date, death_date))
    person_id = cursor.lastrowid

    # Генерируем родственные связи
    if person_id > 1:
        num_relationships = randint(1, min(person_id - 1, 5))  # Ограничиваем количество связей

        for _ in range(num_relationships):
            person2_id = randint(1, person_id - 1)
            relationship_types = ['Parent', 'Child', 'Spouse', 'Sibling', 'Other']
            relationship_type = choice(relationship_types)

            cursor.execute('INSERT INTO Relationships (Person1ID, Person2ID, RelationshipType) VALUES (?, ?, ?)',
                           (person_id, person2_id, relationship_type))

conn.commit()
conn.close()


```
Read.py
```bash
import sqlite3
import pandas as pd

conn = sqlite3.connect('genealogy_tree.db')

people_df = pd.read_sql_query('SELECT * FROM People', conn)
print("Таблица People:")
print(people_df)

relationships_df = pd.read_sql_query('SELECT * FROM Relationships', conn)
print("\nТаблица Relationships:")
print(relationships_df)

conn.close()
```
![image](https://github.com/xanna-ai/Alfa/assets/85250540/0fb5b95e-6849-44c3-ae60-676f6390f4e8)
![image](https://github.com/xanna-ai/Alfa/assets/85250540/fd0f0fae-a9e2-46d7-ac50-62b49d460b7e)

# Задача №2.
Спроектируйте структуру БД для хранения лимитов на суммы платежных операций (минимальный разовый платеж, 
максимальный разовый платеж и максимальная сумма платежей в месяц), совершаемых клиентами через 
удаленные каналы доступа (банкомат, интернет-банк, мобильный банк и т.д.) Под платежными операциями 
понимаются платежи (за мобильный, за интернет и др.), переводы (в сторонний банк, между своими счетами и т.д.). 
пополнение депозита и другие.
### Требования: 
Суммы лимитов должны настраиваться как для отдельной операции (платеж за мобильный), так и 
для группы операций (все переводы со счета клиента)
Суммы лимитов могут отличаться в разных каналах самообслуживания
### Пример.
Через интернет-банк клиент может оплатить мобильный на сумму от 100 до 2000 руб., при этом в сутки 
таких платежей может быть выполнено на сумму не более 2000 руб.
Через банкомат клиент может оплатить мобильный на сумму от 100 до 5000 руб., при этом в день таких 
платежей может быть выполнено на сумму не более 5000 руб.

## Решение
Таблицы и атрибуты для базы данных, которая хранит лимиты на платежные операции:

Таблица "Operations":
* operation_id (PK) - уникальный идентификатор операции
* operation_name - название операции 
  
Таблица "Channels":
* channel_id (PK) - уникальный идентификатор канала самообслуживания 
* channel_name - название канала 
  
Таблица "OperationGroup":
* group_id (PK) - уникальный идентификатор группы операций 
* group_name - название группы операций 
  
Таблица "Limits":
* limit_id (PK) - уникальный идентификатор лимита
* operation_id (FK) - внешний ключ на таблицу "Operations", связь с операцией
* channel_id (FK) - внешний ключ на таблицу "Channels", связь с каналом самообслуживания
* group_id (FK) - внешний ключ на таблицу "OperationGroup", связь с группой операций
* min_single_payment - минимальный разовый платеж
* max_single_payment - максимальный разовый платеж
* max_monthly_payment - максимальная сумма платежей в месяц
![image](https://github.com/xanna-ai/Alfa/assets/85250540/32bca39f-d3c1-485e-83bb-a5bcd03be070)

# Задача №3.
Напишите на псевдокоде (не SQL) алгоритм функции определения возможности совершения платежа по БД, 
спроектированной в предыдущей задаче.

__*Входные параметры:*__
 cумма платежа,
 тип операции,
 канал самообслуживания
 
__*Выходные параметры:*__
 результат проверки ("Y" - платеж разрешен, "N" -- платеж запрещен)
 ## Решение
 ```bash
function check_payment_feasibility(amount, operation_type, channel_type):
    operation = find_operation_by_name(operation_type)
    if operation is None:
        return "N"  

    operation_group = find_operation_group_by_id(operation.group_id)
    if operation_group is None:
        return "N"  

    channel = find_channel_by_name(channel_type)
    if channel is None:
        return "N"  

    limit = find_limit_by_keys(operation.operation_id, operation_group.group_id, channel.channel_id)
    if limit is None:
        return "N"  

    if amount < limit.min_single_payment or amount > limit.max_single_payment:
        return "N"

    total_monthly_payment = get_total_monthly_payment(operation.operation_id, channel.channel_id)
    if total_monthly_payment + amount > limit.max_monthly_payment:
        return "N"

    return "Y"  
 ```

# Задача №4.
Банковская система содержит основную таблицу счетов, в которой хранятся счета всех клиентов банка:
```sql
CREATE TABLE account (
id CHAR NOT NULL, -- идентификатор счета
client_id CHAR NOT NULL, -- идентификатор клиента, владельца счета
created DATE, -- дата открытия счета
closed DATE, -- дата закрытия счета
amount BIGINT, -- баланс по счету
PRIMARY KEY(id)
)
```
Необходимо написать SQL выражения для запросов, возвращающих:
* список открытых счетов клиента client_id=<номер клиента> на сегодняшний день
  ```sql
  SELECT id, client_id, created, closed, amount
  FROM account
  WHERE client_id = <номер клиента> AND closed IS NULL AND created <= CURRENT_DATE;
  ```
* список клиентов, которые закрыли все свои счета
  ```sql
  SELECT DISTINCT client_id
  FROM account
  WHERE closed IS NOT NULL
  GROUP BY client_id
  HAVING COUNT(*) = COUNT(closed);
  ```
* список самых "ценных" для банка клиентов, пояснив, как вы определяете "ценность" клиента.

  Клиенты с наибольшим суммарным балансом будут считаться самыми "ценными" для банка.
  ```sql
  SELECT client_id, SUM(amount) AS total_balance
  FROM account
  WHERE closed IS NULL
  GROUP BY client_id
  ORDER BY total_balance DESC;
  ```
