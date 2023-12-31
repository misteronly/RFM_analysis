# RFM Анализ. Что это и с чем его едят?
![RFM_segments](https://github.com/misteronly/RFM_analysis/blob/main/rfm_picture.png)
### :grey_question: Кто наши лучшие клиенты?
### :grey_question: Кто из клиентов на грани потери?
### :grey_question: На какие сегменты можно разделить наших покупателей?
### :grey_question: На эти вопросы отвечает RFM анализ.
___
**RFM Анализ** - метод, позволяющий сегментировать клиентов, используя прошлое\
покупательское поведение и деление на сегменты.\
Деление пользователей осуществляется на 3 группы:
1. Recency. Как давно клиент совершил покупку у нас - последняя дата покупки
2. Frequency. Как часто делают покупки - количество заказов
3. Monetory. Какую прибыль приносит покупатель - общая сумма

__Зачем нужен RFM Анализ?__

Благодаря разделению пользователей на различные группы, мы сможем \
более точечно подбирать рекламные предложения для каждого сегмента.\
Это позволит улучшить взаимодействие с клиентами, прибылью, стимулировать\
к повторным покупкам, а также вернуть спящих и давних клиентов.
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
/*Считаем разницу в днях между последним заказом и 01.01.2023,
Количество заказов, общую сумму для каждого клиента.
Выбираем дату с начала нового года, чтобы провести аналитику за текущий период*/
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
/*Для RFM анализа необходимо разбить юзеров на 27 когорт. Для этого необходимо выделить
 в каждом сегменте по 3 группы. Для этого, считаем перцентили 33 и 66 для каждой группы.*/
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
    (select (CAST(percentile_cont(0.66) within group(order by monetory) as INT)) as m_perc_66 from group_dataset)
  from
  	group_dataset
),
/*Производим деление клиентов на сегменты:
Чем меньше давность заказа(recency), тем выше оценка.Для частоты(frequency) и
суммы заказов(monetory) соотвественно: высокие значения соотвествуют значительным результатам*/
segment_rfm as (
  select 
  	clients,
  	recency,
  	case 
  		when recency < r_perc_33 then 3
  		when recency < r_perc_66 then 2 else 1
  	end as R,
  	frequency,
  	case 
  		when frequency <= f_perc_33 then 1
  		when frequency <= f_perc_66 then 2 else 3
  	end as F,
  	monetory,
  	case 
  		when monetory <= m_perc_33 then 1
  		when monetory <= m_perc_66 then 2 else 3
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
/*Группируем клиентов на 22 основных сегмента*/
groups_rfm as (
 select 
  	cnt_clients,
  	rfm_full,
  	case 
  		when rfm_full in ('333') then 'VIP'
  		when rfm_full in ('332', '323') then 'Чемпионы - любители'
		when rfm_full in ('331') then 'Чемпионы - новички'
		when rfm_full in ('322') then 'Лояльные постоянники'
		when rfm_full in ('321') then 'Лояльные прохожие'		
		when rfm_full in ('313') then 'Новички транжиры'
		when rfm_full in ('312', '311') then 'Новички экономисты'
		when rfm_full in ('233') then 'Будующие чемпионы'
		when rfm_full in ('232','231') then 'Примерные семьянины' 	
		when rfm_full in ('223','222') then 'Статистические среднячки' 
		when rfm_full in ('221') then 'Любители гематогенов' 
		when rfm_full in ('213') then 'Любители по крупному' 
		when rfm_full in ('212','211') then 'Любители по мелкому'
		when rfm_full in ('133') then 'Потерянные фанаты' 
		when rfm_full in ('132') then 'Подающие надежды' 
		when rfm_full in ('131') then 'Придирчивые'
		when rfm_full in ('123') then 'Бывшие шопоголики'
		when rfm_full in ('122') then 'Залетные птицы'
		when rfm_full in ('121') then 'Из другого района'
		when rfm_full in ('113') then 'Ожидающие скидок'
		when rfm_full in ('112') then 'Случайные прохожие'
		when rfm_full in ('111') then 'На скамейке запасных' 
  	end as segment_rfm
  from
    union_rfm
),
total_clients as (
/*Считаем общую сумму по количеству клиентов для каждого из 22 сегментов */
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
	ROUND(clients /  SUM(clients) over() * 100, 1) as percent_rfm
from
	total_clients
order by percent_rfm desc
```

### Выводы
:white_check_mark: Был произведен анализ покупательской активности, по следующим параметрам: давность, частота покупок, а так же сумма покупок.

:white_check_mark: Произведено разделение датасета на различные сегменты, в зависимости от покупательских особенностей.

:white_check_mark: На основе этих данных, подсчитанно: какое количество пользователей принадлежит каждому сегменту. И выработаны практические действия для активации каждого из 22 сегментов.

__В результате имеем:__
+ доминирующий сегмент - «**VIP**», они составляют 15.2%.
	+ Это наши сливки, нужно показывать, что мы дорожим ими. Например, отправить особые предложения, выдать лимитированные скидочные карты. 
 		+  Например: «Вы – наш главный гость! На следующей неделе у вас будет эксклюзивный доступ к новому приложению для учета здоровья! Теперь вы можете следить за своими показателями и получать персонализированные рекомендации прямо на устройстве! Пожалуйста, подтвердите свое участие, нажав кнопку ниже».	  
+ на 2 месте – «**На скамейке запасных**», они составляют 14.2%
	+ Данная сегмент покупал очень давно и на низкий чек. Скорее всего они разово заходили к нам, поэтому можно отправить обычную рассылку, сосредоточив внимание на более лояльных группах.
 		+ Например: «Вас ждет сюрприз! При покупке товаров на сумму свыше 500р, бонусные баллы удваиваются!».
+ на 3 месте – «**Любители по мелкому**», они составляют 10.8%
	+ Текущая выборка юзерев покупал относительно недавно, на средне-низкий чек. Необходимо предложить им скидку, чтобы стимулировать к покупке.
 		+  Например:«А вы знали, что сегодня Шиповников день?! Вручаем скидку 15% на все товары, в составе которых присутствует полезная ягода.»
+ На 4 месте - «**Статистические среднячки**» они составляют 8%.
	+ Указанная когорта пользователей отличается средними значениями по всем статистическим показателям. Они заходят к нам периодически и покупают на среднюю сумму. Поэтому сделаем предложения, стимулирующее посетить нас в ближайшее время.
 		+  Например: «При покупки двух паст третья в подарок! У нас стартовала продажа новых паст и зубных щеток от итальянский производителей Спешите к нам и дарите блистательные улыбки близким!»
 + На 5 месте - «**Будующие чемпионы**» они составляют 7.9%.
 	+ Данный сегмент представляет собой покупателей, которые совершают покупки относительно часто и на солидную сумму. Необходимо заинтересовать их новыми предложениями.
  		+  Например, «Здравствуйте, Олеся! Помните, здоровье – ваш главный трофей! Дарим скидку в 20% на обновленный ассортимент здоровых перекусов и спортивных БАДов!»
  + На 6 месте –  «**Новички экономисты**», они составляют по 6.5%.
  	+ Текущая группа представляет покупателей, которые совершают покупки часто, покупая товары в средне-низком сегменте. Необходимо сформировать предложения, чтобы они купили на бОльшую сумму.
   		+ Напримр: «Добро пожаловать в мир доступных цен! Позаботьтесь о себе и своих близких со скидкой 25% на товары для зрения, при покупки от 4000р.»
  + На 7 месте –  «**Случайные прохожие**», они составляют по 6.3%.
   	+  Указанная когорта представляет юзеров, которые покупали давно, мало и на средний чек. Отправим им стандартную рассылку и сосредоточим внимание на других группах.
    		+  Например: «Мы ценим каждого клиента, поэтому дарим скидку в 15% на данный ассортимент товаров….»
+ На 8 месте –  «**Залетные птицы**», они составляют по 4.8%.
	+  Эта группа пользователей покупала товары давно, со средней частотой и на среднюю сумму. Необходимо стимулировать их прийти к нам снова.
 		+   Например: «Ваше следующее посещение будет особенным! При покупке товаров на сумму свыше 1500р., дарим скидку 25% на весь ассортимент уходовой косметики!»
+ На 9 месте –  «**Чемпионы любители**», они составляют 4.4%.   
 	+ Данная выборка представляет собою людей, которые покупали недавно, часто и на средне-солидную сумму. Следует показать, что ими дорожат и предложить эксклюзивные предложения.
  		+   Например: «Здравствуйте, первооткрыватели! Для тех кто ценит здоровье, мы сделали новый сервис в приложении – онлайн-консультация врача! Для постоянных клиентов особая цена 690 р. вместо 990р. на первые три записи. Записывайтесь и приводите здоровье в порядок!»
+ На 10 месте –  «**Лояльные постоянники**», они составляют 4.1%.
	+ Заданный сегмент представляет покупателей, которые были у нас недавно и купили на среднюю сумму. Стоит рассказать о новинках и сделать акцент на их важности для нас.
  		+   Например: «Привет! Новинка в нашей аптеке: органические продукты для заботы о коже и волосах. Как лояльному клиенту, вы получается эксклюзивную скидку в 25%!»
+ На 11 месте –  «**Примерные семьянины**», они составляют 2.8%.
	+ Эта группа покупает товары у нас часто, но в относительно низком ценовом диапазоне. Необходимо стимулировать их к покупкам на большую сумму.
  		+   Например: «Дорогой клиент, ваша семья заслуживает лучшего, поэтому на комплекс витаминов от 2500 рублей, вручаем промокод на следующую покупку! »
+ На 12 месте –  «**Любители гематогенов**», они составляют 2.7%.
	+ Данная когорта пользователей отличается средними парметрами по частоте покупок, при этом совершая траты в низком ценовом сегменте. Им стоит предложить товары, которые удовлетворяют их ценовой диапазон.
  		+   Например: «Владимир, вы знали, что чай не только согревает, но и придает сил ? Поэтому специально для вас – подборка травяных чаев из горного Алтая до 500 р.»
+ На 13 месте –  «**Из другого района**», они составляют 2.4%.
	+ Указанный тип покупателей посещали нас давно и производили оплату на низкую сумму. Возможно, это люди проходившие мимо, поэтому предложим им скидку, и сосредоточимся на других группах.
  		+   Например: «Давно вас не видели. Поэтому приглашаем на неделю низких цен и дарим скидку в 15% на определнный ассортимент товаров….»
+ На 14 месте –  «**Потерянные фанаты**», они составляют 2.2%.
	+ Заданный сегмент характеризует пользователей, которые покупали часто на солидную сумму, но по каким-то причинам перестали к нам заходить. Необходимо напомнить, что они важны для нас.
  		+   Например: «Пропали из виду, но не из сердца! Напоминаем, что у вас максимальный уровень карты лояльности! Поэтому специально для вас тройной кэшбэк на все покупки в течении недели!»
+ На 15 месте –  «**Бывшие шопоголики**», они составляют 1.9%.
	+ Указанная выборка имеет показатели по возрастающей: покупки были давно, частота была средняя, сумма солидная. Необходимо подцепить их подтверждением емайла, чтобы отправлять выгодные предложения.
  		+   Например: «Мы ценим каждого клиента! Подтвердите свой почтовый адрес и получите промокод на онлайн заказ: 700р. при покупке от 3000р.! Акция действует в течении недели»
+ На 16 месте –  «**Лояльные прохожие**», они составляют 1.7%.
	+ Текущаю группа юзеров имеет показатели по убывающей: последняя покупка была недавно, заказывают относительно часто, при этом чек находится в низком ценовом сегменте. Стоит сделать акцент на полезном дополнении при покупки от нужной нам суммы.
  		+   Например: «Привет! При покупке товаров свыше 2000 рублей вам полагается подарок: набор для ухода за телом!»
+ На 17 месте –  «**Подающие надежды**», они составляют 1.1%.
	+ Заданный сегмент клиентов совершали покупки давно, при этом покупали часто и на среднюю сумму. Возможно они ожидают товары, которые раньше охотно покупали. Предложим скидку на часто покупаемую продукцию.
  		+   Например: «Здравствуйте, мы скучаем по вам! Только до 1 марта действует супер-скидка 20% на вашу любимую категорию товаров!
+ Также на 17 месте –  «**Ожидающие скидок**», они составляют 1.1%.
	+ Указанная кагорта пользователей осуществляли покупки давно, покупали мало, но на крупную сумму. Необходимо предложить акцию, чтобы они вернулись за покупкой.
  		+   Например: «На следующих выходных у нас пройдет акция – Здоровье в каждый дом! При покупке товаров для всей семьи на сумму 3000р, вы получаете экономию в 20%.»
+ На 18 месте –  «**Любители по крупному**», они составляют 0.9%.
	+ Данный сегмент пользователей осуществляли покупки относительно недавно, и на среднюю сумму. Необходимо напомнить о нашей карте лояльности, чтобы они вернулись за покупкой.
  		+   Например: «На вашей карте уже 2400 бонусов! Мы гордимся вами! Успейте потратить их до конца месяца, пока не истек срок давности.»
+ На 19 месте –  «**Новички транжиры**», они составляют 0.5%.
	+ Текущая группа юзеров покупали у нас недавно, на высокую сумму, но заглядывают к нам редко. Следует отправить выгодное предложение,чтобы они заходили чаще.
  		+   Например: «Приветствуем, новый друг! Для вас выгодное предложение: 15% на следующие три покупки в нашей аптеке!»
+ На 20 месте –  «**Чемпионы новички**», они составляют 0.2%.
	+ Указанная выборка покупателей, производили траты недавно, приходят к нам часто, но сумма заказа низкая. Нужно предложить скидку, но начиная от определенной цены.
  		+   Например: «Здравствуйте, спешим вас обрадовать, скидка 500р. при покупке от 3000р. У нас обновление ваших любимых товаров! Новые витамины на все случае жизни. Мы заботимся о вас, чтобы вы позаботились о себе!»
+ На 21 месте –  «**Придирчивые**», они составляют 0.1%.
	+ Текущий тип покупателей является самым наименьшим из всех вышепредставленных. Совершают покупки редко и на низкую сумму, скорее всего у них присутсвуют свои предпочтения. Попытаемся расположить их к себе.
  		+   Например: «Мы учли ваши особенные запросы и приготовили подарок! При покупке любых товаров на сумму 2500р. вы получаете эксклюзивный набор от наших партнеров!»
