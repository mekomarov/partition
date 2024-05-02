# partition
Секционирование таблицы
```
--Для выполнения задания по секционированию накатил на локальный PostgreSQL БД демо-биг

--ticket_flights
select relname,relpages,*
from pg_class
order by relpages desc
limit 10

--Выбор атрибута для секционирования
select *
from pg_stats
where tablename = 'ticket_flights'


--По нашему ТЗ ежедневно нужно предоставлять 3 разных отчета по каждому классу обслужтвания (Бизнес, комфорт и эконом)
--копируем структуру 
create table bookings.ticket_flights_part (like bookings.ticket_flights) partition by list (fare_conditions);
--Создаем партиции 
create table bookings.ticket_flights_economy partition of bookings.ticket_flights_part for values in ('Economy');
create table bookings.ticket_flights_comfort partition of bookings.ticket_flights_part for values in ('Comfort');
create table bookings.ticket_flights_business partition of bookings.ticket_flights_part for values in ('Business');
--Вставляем данные в секционированную таблицу
insert into bookings.ticket_flights_part(ticket_no, 
										 flight_id, 
										 fare_conditions, 
										 amount)
select ticket_no, 
	   flight_id, 
	   fare_conditions, 
	   amount
from bookings.ticket_flights;

--Проверяем производительность запросов в сравнении с секционированной таблицей

--Задание 1 Вывести суммы по билетам в классе обслуживания эконом
--Комментарии:
/*
В таком варианте секционирование не ускорило время выполнения запроса по условии класс обслуживания Эконом, 
т.к. большая часть билетов именно этого класса, а то пересбора статистики еще 
и выполнялся ддольше из плана показал время выполнения    
*/

--20 секунд
--Не секционированная таблица
explain analyze
select fp.ticket_no,
	   sum(fp.amount) as total_amount  
from bookings.ticket_flights as fp
where 1=1 
and fp.fare_conditions = 'Economy'
group by fp.ticket_no;

--1 минута 6 секунд
--Секционированная
explain analyze
select fp.ticket_no,
	   sum(fp.amount) as total_amount  
from bookings.ticket_flights_part as fp
where 1=1 
and fp.fare_conditions = 'Economy'
group by fp.ticket_no;

--Пересоберем статистику по таблице
analyze verbose bookings.ticket_flights_part; 

--20 секунд 
--Секционированная
explain analyze
select fp.ticket_no,
	   sum(fp.amount) as total_amount  
from bookings.ticket_flights_part as fp
where 1=1 
and fp.fare_conditions = 'Economy'
group by fp.ticket_no;

--2 Задание Вывести суммы по билетам в классе обслуживания Business
/*
 * Чтение таблицы происходит построчное в запросах, только в во втором запросе сканирование идет только по секции  
 */

--3.8 Секунды
explain analyze
select fp.ticket_no,
	   sum(fp.amount) as total_amount  
from bookings.ticket_flights as fp
where 1=1 
and fp.fare_conditions = 'Business'
group by fp.ticket_no;

--1.9 секунды
explain analyze
select fp.ticket_no,
	   sum(fp.amount) as total_amount  
from bookings.ticket_flights_part as fp
where 1=1 
and fp.fare_conditions = 'Business'
group by fp.ticket_no;




