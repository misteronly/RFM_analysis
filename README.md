# RFM_analysis 
![ff](https://www.google.com/search?q=rfm+%D1%82%D1%84%D0%B4%D0%BD%D1%8B%D1%88%D1%8B&tbm=isch#imgrc=kGVcnseGwyDJXM)
### :grey_question: Кто наши лучшие клиенты?
### :grey_question: Кто из клиентов на грани потери?
### :grey_question: На какие сегменты можно разделить наших покупателей?
### :grey_question: На эти вопросы отвечает RFM анализ.
___
**RFM Анализ** - метод, позволяющий сегментировать клиентов, используя прошлое\
покупательское поведение и деление на сегменты
Деление RFM проиходит на 3 группы:
1. Recency. Как давно клиент совершил покупку у нас - последняя дата покупки
2. Frequency. Как часто делают покупки - количество заказов
3. Monetory. Какую прибыль приносит покупатель - общая сумма

__Зачем нужен RFM Анализ?__

Благодаря разделению пользователей на различные группы,\
мы сможем более точечно подбирать рекламные предложения для каждого сегмента.\
Это позволит улучшить взаимодействие с клиентами, прибыль, стимулирование к\
повторным покупкам, а также вернуть спящих и давних клиентов.
```sql
/*Выбираем поля для расчетов: клиент, дата, сумма чека*/
with dataset as (
  select
  	card as clients,
  	date(datetime) as datetime,
  	summ  
  from
  	bonuscheques b 
),
/*Считаем разницу в днях между последним заказом и 01.01.2023
Количество заказов
Общую сумму для каждого клиента*/
group_dataset as (
  select
  	clients,
  	CAST('2023-01-01' as date) - MAX(datetime) as recency,
  	COUNT(summ) as frequency,
  	SUM(summ) as monetory
  from 
  	dataset
  group by clients
),
/*Считаем перцентили 33 и 66 для каждой группы*/
perc_rfm as (
  select 
    clients,
    recency,
    frequency,
    monetory,
    (select percentile_cont(0.33) within group(order by recency) as r_perc_33 from group_dataset),
    (select percentile_cont(0.66) within group(order by recency) as r_perc_66 from group_dataset),
    (select percentile_cont(0.33) within group(order by frequency) as f_perc_33 from group_dataset),
    (select percentile_cont(0.66) within group(order by frequency) as f_perc_66 from group_dataset),
    (select percentile_cont(0.33) within group(order by monetory) as m_perc_33 from group_dataset),
    (select ROUND(CAST(percentile_cont(0.66) within group(order by monetory) as INT), 2) as m_perc_66 from group_dataset)
  from
  	group_dataset
),
/*Производим деление клиентов на сегменты:
Чем меньше давность заказа(recency), тем выше оценка. Для частоты(frequency) и
суммы заказов(monetory) наоборот: высокие значения соотвествуют низким резульататам*/
segment_rfm as (
  select 
  	clients,
  	recency,
  	case 
  		when recency < r_perc_33 then 1
  		when recency < r_perc_66 then 2 else 3
  	end as R,
  	frequency,
  	case 
  		when frequency <= f_perc_33 then 3
  		when frequency <= f_perc_66 then 2 else 1
  	end as F,
  	monetory,
  	case 
  		when monetory <= m_perc_33 then 3
  		when monetory <= m_perc_66 then 2 else 1
  	end as M 
  from
  	perc_rfm
),
/*Считаем кол-во клиентов для разных RFM - групп*/
union_rfm as (
  select
  	concat_ws('',r,f,m) as rfm_full,
  	COUNT(clients) as cnt_clients
  from
    segment_rfm
  group by concat_ws('',r,f,m)
),
/*Группируем клиентов на 8 основных сегментов*/
groups_rfm as (
  select 
  	cnt_clients,
  	rfm_full,
  	case 
  		when rfm_full in ('111','112','113','121','131') then 'Чемпионы'
		when rfm_full in ('212','211','122','123','132','133') then 'Растущие'
		when rfm_full in ('223','312','313','322','231','331','221','311','321') then 'На грани'
		when rfm_full in ('213','222','232','233') then 'Новички'
		when rfm_full in ('333','332','323') then 'Спящие'		
  	end as segment_rfm
  from
    union_rfm
),
total_clients as (
/*Считаем общую сумму по количеству клиентов для каждого из 8 сегментов */
	select
		segment_rfm,
		SUM(cnt_clients) as clients
	from
	  groups_rfm
	group by segment_rfm
	order by clients DESC
)
/*Считаем долю в процентах каждого сегмента*/
select 
	segment_rfm,
	ROUND(clients /  SUM(clients) over() * 100, 0) as percent_rfm
from
	total_clients
order by percent_rfm desc
```

### Выводы
:white_check_mark: Был произведен анализ пользователей по давности, количеству, а так же сумме покупок.

:white_check_mark: Произведено разделиление датасета на различные сегменты и вычисление доли относительно общего числа клиентов

:white_check_mark: На основе этих данных, подсчитанно: какое количество пользователей принадлежит каждому сегменту.

__В результате имеем:__
+ доминирующий сегмент - это спящие, они составляют 23%.
  + Данному сегменту нужно: предложить релеватные продукты, а также специальные предложения и скидки.
+ на 2 месте - растущие, они состалвяют 22%
  + Для данного сегмента, необходимо предложить бесплатные тестеры, различные триал периоды
+ на 3 месте - на грани, они составляют 19%
  + Для них нужно: делиться полезной информацией, рекомендовать популярные товары и услуги, предлагать промокоды
+ 4 место делят две группы - чемпионы и новички, они составляют 18%
  + Первую подгруппу необходимо награждать, отправлять особые предложения
  + Для второй подгруппы необходимо: создать устойчивое понимание бренда. Привлечь акциями, чтобы клиент понял пользу нашего продукта

