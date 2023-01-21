--1. Сколько оплатил каждый пользователь за прокат фильмов за каждый месяц
```sql
select distinct c.customer_id,
	   date_part('year', p.payment_date) "year",
	   date_part('month', p.payment_date) "month", 
	   sum(p.amount) over (partition by c.customer_id, date_part('month', p.payment_date))
from customer c
	join payment p on p.customer_id = c.customer_id
order by 1```


-- 2. На какую сумму продал каждый сотрудник магазина
```sql
select distinct s.staff_id, sum(p.amount) over (partition by s.staff_id)
from staff s
	join payment p on p.staff_id = s.staff_id
order by 1```
	

--3. Сколько каждый пользователь взял фильмов в аренду
```sql
select distinct c.customer_id "Пользователь",
	   count(f.film_id) over (partition by c.customer_id) "Количество фильмов"
from customer c
	join rental r on r.customer_id = c.customer_id
	join inventory i on i.inventory_id = r.inventory_id
	join film f on f.film_id = i.film_id
order by 1```


--4. Сколько раз брали в прокат фильмы, в которых снимались актрисы с именем Julia
```sql
select count(*)
from rental r
	join inventory i on i.inventory_id = r.inventory_id 
	join film f on f.film_id = i.film_id
	join film_actor fa on fa.film_id = f.film_id
	join actor a on a.actor_id = fa.actor_id
where a.first_name ilike 'Julia'```


--5. Сколько актеров снимались в фильмах, в названии которых встречается подстрока bed
```sql
select count(distinct(fa.actor_id)) "Количество актеров"
from film f
	join film_actor fa on fa.film_id = f.film_id
where fulltext::text like '%''bed''%'```


--6. Вывести пользователей, у которых указано два адреса
```sql
select c.customer_id
from customer c 
	join address a on a.address_id = c.address_id
where a.address2 is not null```


--7. Сформировать массив из категорий фильмов и для каждого фильма вывести индекс массива соответствующей категории
```sql
select f.title,
       c."name",
       array_position((select array_agg (name) from category), c."name")
from film f 
	join film_category fc on fc.film_id = f.film_id
	join category c on c.category_id = fc.category_id```


--8. Вывести массив с идентификаторами пользователей в фамилиях которых есть подстрока 'ah'
```sql
select array_agg (customer_id)
from customer c
where last_name ilike '%ah%'```


--9. Вывести фильмы, у которых в названии третья буква 'b'
```sql
select title
from film f
where title ilike '__b%'```


--10. Найти последнюю запись по пользователю в таблице аренда без учета last_update
```sql
select customer_id, rental_date
from
	(select customer_id,
		    rental_date,
		    row_number() over (partition by customer_id order by rental_date desc) 
	from rental r) t
where row_number = 1```


--11. Вывести ФИО пользователя и название третьего фильма, который он брал в аренду.
```sql
select customer, title
from
	(select c.first_name || ' '|| c.last_name "customer",
		    f.title,
		    row_number() over (partition by c.customer_id order by r.rental_date)
	from customer c 
		join rental r on r.customer_id = c.customer_id
		join inventory i on i.inventory_id = r.inventory_id
		join film f on f.film_id = i.film_id) t 
where row_number = 3```


--12. Вывести пользователей, которые брали один и тот же фильм больше одного раза.
```sql
select distinct(id)
	from (select c.customer_id id, f.film_id, count(*) 
	from customer c
		join rental r on r.customer_id = c.customer_id
		join inventory i on i.inventory_id = r.inventory_id
		join film f on f.film_id = i.film_id
	group by 1,2
	having count(*) > 1) t
	order by 1```


--13. Какой из месяцев оказался самым доходным?
```sql
select "month" from
	(select distinct date_part('month', p.payment_date) "month",
		   sum(amount) over (partition by date_part('month', p.payment_date))
	from payment p
	order by 2 desc
	limit 1) t```
	

--14. Одним запросом ответить на 2 вопроса: в какой из месяцев взяли в аренду фильмов больше всего? На сколько по отношению к предыдущему месяцу
-- было сдано в аренду больше/меньше фильмов.
	```sql
with cte as (select distinct date_part('month', r.rental_date) "month",
	   		    count(r.rental_id) over (partition by date_part('month', r.rental_date))
			 from rental r
			 order by 1)
select month, count - lag(count) over (order by cte.month) "difference"
from cte
order by count desc 
limit 1```



--15. Определите первые две категории фильмов по каждому пользователю, которые они чаще всего берут в аренду.
```sql
select customer_id, array_agg(name) "categories" from
	(select *, row_number() over (partition by customer_id) from 
		(select c.customer_id, c2.name, count(*)
		from customer c 
			join rental r on r.customer_id = c.customer_id
			join inventory i on i.inventory_id = r.inventory_id
			join film f on f.film_id = i.film_id 
			join film_category fc on fc.film_id = f.film_id
			join category c2 on c2.category_id = fc.category_id
			group by 1,2
		order by 1, 3 desc) t) t
where row_number in (1,2)
group by 1```



--БД bookings

--1. Сколько суммарно каждый тип самолета провел в воздухе, если брать завершенные перелеты.
```sql
select distinct a.model,
	   sum(f.actual_arrival - f.actual_departure) over (partition by a.model) "sum_duration"
from aircrafts a
	join flights f on f.aircraft_code = a.aircraft_code
where f.actual_arrival is not null```


--2. Сколько было получено посадочных талонов по каждой брони
```sql
select distinct b.book_ref, count(bp.boarding_no) over (partition by b.book_ref)
from boarding_passes bp
join tickets t on t.ticket_no = bp.ticket_no
join bookings b on b.book_ref = t.book_ref```


--3. Вывести общую сумму продаж по каждому классу билетов
```sql
select distinct fare_conditions,
	   sum(amount) over (partition by fare_conditions)
from ticket_flights tf```


--4. Найти маршрут с наибольшим финансовым оборотом
```sql
select flight_no from
(select distinct f.flight_no,
	    sum(tf.amount) over (partition by f.flight_no)
from flights f 
	join ticket_flights tf on tf.flight_id = f.flight_id
order by 2 desc) t
limit 1```


--5. Найти наилучший и наихудший месяцы по бронированию билетов (количество и сумма)
```sql
with c as (select distinct date_part('month', book_date) "month",
	   sum(total_amount) over (partition by date_part('month', book_date)),
	   count(total_amount) over (partition by date_part('month', book_date))
	   from bookings
	   order by sum desc
	   limit 1),
	 c2 as (select distinct date_part('month', book_date) "month",
	   sum(total_amount) over (partition by date_part('month', book_date)),
	   count(total_amount) over (partition by date_part('month', book_date))
	   from bookings
	   order by sum
	   limit 1)
select c.month "best_month", c2.month "worst_month"
from c, c2```
