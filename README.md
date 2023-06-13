## Домашнее задание №7

1.	Настройте выполнение контрольной точки раз в 30 секунд.
> sudo -u postgres psql
> show checkpoint_timeout;
> alter system set checkpoint_timeout = '30s';
> 
-- Перезагрузим постгре и проверим результат:

> sudo systemctl restart postgresql
> show checkpoint_timeout;

![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/b4a46a35-5549-4008-8a56-1ea4eeb41da3)

 

2.	10 минут c помощью утилиты pgbench подавайте нагрузку.
Проверим объем журнальных файлов до:

> sudo du -h /var/lib/postgresql/15/main/pg_wal
> sudo -u postgres pgbench -i postgres

![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/086cc4fa-26ec-4649-92d5-9dc73f94ca0b)

 
> sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/a4b1278b-aa00-4eb2-8cd2-d6dd752dce4d)


3.	Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

> sudo du -h /var/lib/postgresql/15/main/pg_wal
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/76b044b4-0804-43fd-b19d-5b13a673cdcc)


> Получается, что на одну контрольную точку приходится (65 – 17) / 20 = 48 / 20 = 2,4 мегабайта данных.

4.	Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

> tail -n 50 /var/log/postgresql/postgresql-15-main.log
 
![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/df20366a-2807-4e84-84a3-0db8fd8d9a3d)


> При выполнении нагрузочного тестирования видим, что все контрольные точки были пройдены. Судя по статистике среднее время выполнения контрольной точки ~27 секунд. Поскольку база была пустой, точки не накладывались друг на друга. При большей нагрузке нет смысла так часто делать контрольные точки.

5.	Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
Убедимся, что включен синхронный режим и выполним замер производительности:

> show synchronous_commit;

 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/a62761c0-6a31-4aa6-b3f7-f01cfbddcc62)


Выключим синхронный режим и перезагрузим постгре:

> alter system set synchronous_commit = 'off';
> 
> sudo systemctl restart postgresql
> 
> show synchronous_commit;
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/261cb1be-e2e1-4e6f-b832-6250e761ddd5)


Выполним замер производительности:
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/ad8d723a-5419-4d8a-8b4f-53602596ae44)


> В синхронном режиме tps ~ 553, в асинхронном режиме tmp ~ 955. Прирост составил ~73%. Причина в более эффективном сбрасывании на диск, теперь за вместо того что бы писать на диск на каждый чих, мы делаем это отдельным процессом, по расписанию и пачками.

6.	Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

Создадим «test_cluster», сразу включив контроль сумм, и запустим класстер:

> sudo pg_createcluster 15 test_cluster -- --data-checksums
> 
> sudo pg_ctlcluster 15 test_cluster start
> 
> sudo pg_lsclusters
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/766be5a5-01e7-4d28-84b2-9f6345f0acd9)


По умолчанию контроль суммы страниц был включен, убедимся в этом, после чего создадим табличку и заполним ее некоторыми данными.

> sudo -u postgres psql -p 5433
> 
> show data_checksums;
> 
> create table test_tab (name text, age integer);
> 
> insert into test_tab (name, age) values ('ivan', 20);
> 
> insert into test_tab (name, age) values ('sergey', 37);
> 
> insert into test_tab (name, age) values ('anna', 18);
> 
> select * from test_tab ;
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/119e1aa5-dc41-4b72-b25e-f504ae10f49e)


Остановим сервер и поменяем несколько байтов в странице (сотрем из заголовка LSN последней журнальной записи):

> SELECT pg_relation_filepath(' test_tab ');
> 
> sudo pg_ctlcluster 15 test_cluster stop
> 
> sudo dd if=/dev/zero of=/var/lib/postgresql/15/test_cluster/base/5/16384 oflag=dsync conv=notrunc bs=1 count=8
> 
> sudo pg_ctlcluster 15 test_cluster start
> 
> sudo -u postgres psql -p 5433
> 
> select * from test_tab;
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/ce0b945d-4b6c-4ec5-934e-0d225c06cf6f)


> data_checksums гарантирует целостность на уровне байтов в файлах, именно поэтому мы получили сообщение о том, что файл был поврежден

Есть несколько вариантов решения данной проблемы:
1. найти и удалить поврежденные строки (если потеря не критична);
2. восстановить поврежденные строки из бэкапа (при наличии);
3. в текущей сессии проигнорировать ошибку;
4. занулить строки (установить параметр zero_damaged_pages = on) и выполнить полный вакуум.

Попробуем проигнорировать ошибку (способ 3). В моем случае все строки отобразились:
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/510303d2-6543-4ba9-970b-5d12e5f1f446)


Повторим эксперимент и попробуем восстановить данные способом №4. В моем случае не удалось: 

> show zero_damaged_pages;
> 
> set zero_damaged_pages = on;
> 
> select * from test_tab;
 
 ![image](https://github.com/blaidex2/Postgres_Homework-7/assets/130083589/aa98bf04-91ae-4ddf-926a-8e86275afb6f)

