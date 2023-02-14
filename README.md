# pyPhraseSuggesting
Backend implementation for autocomplete form fields.
Поиск окончания фразы. Для автозаполнения в текстовых полях форм.

По введенной строке (предполагается, что последнее слово не окончено) 
ищет несколько наиболее вероятных фраз с оконченным последним словом.

В качества источника фраз используем корпус лента.ру. В последствии надо 
сделать обучающийся механизм, который будет добавлять в словарь фразы, 
введенные самими пользователями в полях формы.

В подборе базируюсь на эту теорию:
https://habr.com/ru/post/675218/

1) Формируем униграмы и биграммы с кол-вом включений, используя паттер map-reduce (mr4mp). 
	зы: вылезла MemoryError при попытке весь корпус загрузить в 24 потока, 
	если убрать колво потоков до 6-8 - всё нормально.

2) Загружем в базу в две коллекции
	1) по униграмам с индексом, по которому будем искать на полное совпадение или по префиксу (/^prefi/) 
		и иметь быструю сортировку по кол-ву включений
	2) по биграмам (используем вместо текста идентификаторы униграм?), индексы по обоим полям
		для поиска и кол-во для сортировки

Так ищем варианты:
На входе: фраза состоящая из введенных слов(или ничего, заканчивается пробелом), 
	последнего неполного слова (или ничего, не заканчивается пробелом).
1) Вычищам, приводим и разбиваем фразу на части - массив слов, последнее неполное слово.
2) Вычитываем в униграммах имеющиеся полные слова, считаем вероятность для фразы - она постоянное основание
3) Подбираем наивероятнейшие точные варианты, проход вперёд, 
	когда слева всё определено и фиксированно, 
	а значит не влияет на правую часть дальше одного шага:

	1) Ищем в биграмах по последнему слову
	2) Ищем по неполному слову в униграммах как префикс, топ-100
	3) Мёржим 1 и 2, оставляя совпадения, получая первичный набор вероятностей
	4) Для каждого варианта неполного слова ищем биграмы, где это слово слева. 
	5) Добавляем к нашему списку цепочек найденное, вытеснительно в рамках лимита.
	6) Делаем 4 пункт пока:
		- не начнется повтор слов в цепочках
		- не дойдём до "_" (конца)
		- не закончатся варианты
		- не достигнем глубины 4
		- вероятности не станут совсем мизерными (порог по медиане уже имеющихся, например)

4) Подбираем наивероятнейшие точные варианты, проход назад,
	когда при введенной фразе предполагаем, что это окончание
	более длинной фразы.

	1) Ищем для левого слова биграммы наиболее вероятные для него, как второго слова
	2) Считаем вероятности цепочек, откидываем не поместившиеся
	3) Повторяем пока:
		- не начнеся повтор
		- не дойдём до _
		- не достигнем глубины 4
		- вероятности не станут совсем мизерными (порог по медиане уже имеющихся, например)


