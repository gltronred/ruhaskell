---
author: Вершилов Александр
title:  Биномиальные очереди как вложенные данные
tags: types, algorithm, translation, okasaki, structures
hrefToOriginal: http://okasaki.blogspot.ru/2009/10/binomial-queues-as-nested-type.html
description: Перевод интересной статьи об использовании вложенных типов из далекого 2009 года.
---

Моисей Котович (Maciej Kotowicz) недавно задал вопрос в спискe
рассылки [Haskell Cafe](https://mail.haskell.org/pipermail/haskell-cafe/), о том как реализовать биномиальные очереди
(так же известные как биномиальные кучи) используя хитрые
возможности системы типов так, чтобы система проверки корректности типов (type-checker)
могла проверить все инварианты формы структуры данных. Этот вопрос напомнил мне о
дискуссии с коллегой летом 1998 года про использование вложенных типов для
гарантии соблюдения инвариантов формы. Немного порывшись в архивах,
я нашёл это письмо, которое и представляю здесь.




Я поиграл с биномиальными очередями как вложенным типом данных,
и я думаю, что вы найдете результат интересным.

## Обзор

Позвольте мне сначала дать краткий обзор обычной реализации
биномиальнух очередей. Вспомним, что биномиальное дерево имеет
следующую форму:

~~~~haskell
data Tree a = Node a [Tree a]
~~~~

У биномиального дерева ранга $k$ есть $k$ детей рангов $k-1 \ldots 0$
сохраненных в списке с в порядке убывания ранга. Заметьте, что мы
можем объединять два пирамидально-упорядоченных (heap-ordered) биномиальных
дерева ранга $k$, получая пирамидально-упорядоченное дерево ранга $k+1$ следующим образом:

~~~~haskell
combine a@(Node x xs) b@(Node y ys)
   | x <= y    = Node x (b : xs)
   | otherwise = Node y (a : ys)
~~~~

Далее, биномиальная очередь это список пирамидально-упорядоченных биномиальных деревьев
в порядке возрастания рангов *[переводчик: тут было высоты (height)]* (но не обязательно последовательных рангов).
Для хранения рангов деревьев, которые присуствуют в структуре, мы будем
использовать их позиции в списке. Как следствие, в списке будут присутсвовать
пустые ранги. Данная структура может быть реализована следующим образом:

~~~~haskell
type Binom a = [Maybe (Tree a)]
~~~~

или более эффективно

~~~~haskell
data Binom a = Nil
             | Zero (Binom a)
             | One (Tree a) (Binom a)
~~~~

или ещё лучше распаковав `Tree` в конструкторе `One`

~~~~haskell
data Binom a = Nil
             | Zero (Binom a)
             | One a [Tree a] (Binom a)
~~~~

Я не буду рассматривать все операции -- они подробно рассмотрены в других источниках.
Я просто опишу две функции. Первая, `add`, берёт дерево и список (где ранг дерева
имеет тот же ранг, что и первый элемент в списке), и возвращает новый список.
Эта операция является аналогом функции инкремента на бинарных числах.

~~~~haskell
add :: Ord a => a -> [Tree a] -> Binom a -> Binom a
add x xs Nil = One x xs Nil
add x xs (Zero h) = One x xs h
add x xs (One y ys h)
  | x <= y    = Zero (add x (Node y ys : xs) h)
  | otherwise = Zero (add y (Node x xs : ys) h)
~~~~

Функция объедиения (`merge`) записывается похожим образом и
работает так же как и функция сложения бинарных чисел.

Наконец, функция `getMin` возвращает минимальный элемент в очереди
и очередь без этого элемента. Вспомогательная фунция `getMin_`
возвращает тройку: минимальный элемент в одном из суффиксов (?) очереди,
список детей, связанных с этим минимальным элеменом, и суффикс без этого элемента.

~~~~haskell
getMin_ :: Ord a => Binom a -> (a, [Tree a], Binom a)
getMin_ (Zero h) = case getMin_ h of
                     (y,ys,h') -> (y,ys,Zero h')
getMin_ (One x xs Nil) = (x,xs,Nil)
getMin_ (One x xs h) = case getMin_ h of
                         (y,ys,h') | x <= y    -> (x,xs,Zero h)
                                   | otherwise -> (y,ys,One x xs h')

getMin :: Ord a => Binom a -> (a, Binom a)
getMin h = let (x,xs,h') = getMin_ h
           in (x,merge (list2binom xs Nil) h')

list2binom [] h = h
list2binom (Node x xs : ys) h = list2binom ys (One x xs h)
~~~~

Заметьте, что когда `getMin` получает список дочерних элементов, она
превращает их в правильную биномиальную очередь, разворачивая список  и
заменяя каждую пару конструкторов `Node` и `(:)` на конструктор `One`).

## Вложенное представление - первая попытка.

Следуя тем же путём, который мы использовали для проектирования
других вложенных типов данных, мы получаем следующее представление биномиальных
очередей.

~~~~haskell
data Binom_ a b = Nil
                | Zero (Binom_ a (Trees a b))
                | One a b (Binom_ a (Trees a b))

type Trees a b = (a, b, b)

type Binom a = Binom_ a ()
~~~~

Например, список детей

~~~~haskell
  (Node x3 xs3 : Node x2 xs2 : Node x1 xs1 : Node x0 xs0 : [])
~~~~

будет представлен вложенными тройками

~~~~haskell
  (xs3,xs3', (x2,xs2', (x1,xs1', (x0,xs0', ()))))
~~~~

(где $xsN'$ - соотвествующее преобразование $xsN$)

Все функции записываются очень просто, за исключением `getMin` и `getMin_`
которые должны инкрементально строить функцию разворота, для того, чтобы
превратить

~~~~haskell
  (xs3,xs3', (x2,xs2', (x1,xs1', (x0,xs0', ()))))
~~~~

в

~~~~haskell
  One x0 xs0' (One x1 xs1' (One x2 xs2' (One x3 xs3' Nil)))
~~~~

В результате того, что данная функция строится инкрементально,
реализация оказывается примерно на 10% медленнее, чем оригинальная.

## Перерыв о развороте списков

Предположим, что мы хотим использовать следующий тип данных. Последовательности 
с тремя операциями:

~~~~haskell
empty :: ReversableSeq a
cons  :: a -> ReversableSeq a -> ReversableSeq a
rev   :: ReversableSeq a -> [a]
~~~~

с очевидной семантикой: `const` должен иметь сложность `O(1)`, но `rev` может
может иметь сложность `O(n)`. Разумной реализацией может быть список со
следующими операциями:

~~~haskell
type ReversableSeq a = [a]
empty = []
cons = (:)
rev = reverse
~~~

но, если вы повнимательнее посмотрите на него, то увидите, что тут получается достаточно
много дополнительных накладных расходов на сопоставление с образцом (pattern matching), которые `reverse`
выполняет на каждом шагу. Ещё в давнем 1985 году, John Hughes придумал представление,
которое убирает эти расходы

~~~~haskll
type ReversableSeq a = [a] -> [a]
empty = id
cons x xs = xs . (x:)
rev xs = xs []
~~~~

Результатом `cons 1 (cons 2 (cons 3 empty))` будет функция

~~~~haskell
id . (3:) . (2:) . (1:)
~~~~

которая будучи применена к `[]` функцией `rev` возвращает `[3,2,1]`. Один из
вариантов представить как работает эта структура, это представить список с "дырками" в концах, `cons`
заполняет эту дырку элементом и другой дыркой, а операция `rev` заполняет
дырку `[]`. (Это структура похожа на разностныме списки из логического программирования...)

Применяя этот же трюк к обычному представлению биномиальных очередей
мы получаем представление

~~~~haskell
data Binom a = Nil
             | Zero (Binom a)
             | One a (Binom a -> Binom a) (Binom a)
~~~~

которое оказывается примерно на 10% быстрее оригинального.

## Вложенное представление -- вторая попытка

Ok, давайте попробуем сделать вложенный тип данных. Концептуально
вложение должно следить за нашей позицией в очереди так, что мы
будем знать ранг текущего дерева, и мы не можем смешать деревья
разных рангов. Мы строим гипотезу о некоем базовом типе обозначающем
начало списка, и тип-преобразование `Succ`, который модифицирует тип
одновременно с тем, как мы движемся вниз по очереди.

~~~~haskell
type Base = ???
type Succ b = ???  -- this might need to be Succ a b
~~~~

~~~~haskell
type Binom a = Binom_ a Base

data Binom_ a b = Nil
                | Zero (Binom_ a (Succ b))
                | One a (Binom_ a ??? -> Binom_ a ???) (Binom_ a (Succ b))
~~~~

Теперь, какие типы мы должны подставить в конструктур `One`? Так.. функция `(Binom_ a ??? ->  Binom_ a ???)`
описывает дочерние элементы текущего узла. Если у текущего узла ранг $k$ то, эти элементы имеют ранги $0 .. k-1$.
Функция `(Binom_ a ??? -> Binom_ a ???)` получает очередь, начинающуюся на ранге $k$, и добавляет
детей перед ним, возвращая очередь, начинающуюся с ранга $0$. Т.о. `???` может быть заполнено
как

~~~~haskell
                | One a (Binom_ a b -> Binom_ a Base) (Binom_ a (Succ b))
~~~~

или просто

~~~~haskell
                | One a (Binom_ a b -> Binom a) (Binom_ a (Succ b))
~~~~

Теперь, какие конструкторы должны быть у `Base` и `Succ`? Это не важно, поскольку мы никогда не будем
строить данные этих типов, мы просто используем эти типы, чтобы описать позицию в очереди.
Поэтому мы можем определить `Base` и `Succ` как

~~~~haskell
type Base = Void
newtype Succ b = S Void
~~~~

Мне кажется, что это очень интересные типы по следующим причинам:

Они включают стрелочный тип (arrow type), который мы не часто встречаем.
В правой части определения типа данных (на самом деле только в конструкторе `One`)
есть входжения `Binom_` не менее чем трех различных типов

~~~~haskell
Binom_ a b
Binom_ a Base
Binom_ a (Succ b)
~~~~

Я видел, когда встречались два вхождения, но не три.
Эти типы никогда не используются, а служат для поддержания инвариантов (в этом случае того,
что три различные ранга не должны быть перепутаны). [Комментарий от 2009года: сегодня
такие штуки называются фантомными типами.]. В определенном смысле это делает типы менее
интересными, но приводят к интересному свойству, что код отвечает свойству удаления типов ("type erasure"):
если вы удалите типы из всех функций, то получите такой же код как и для оптимизированного обычного представления..

Благодаря этому свойству удаления типов, вы можете ожидать, что вы получите код настолько
же быстрый как и для обычного оптимизированного представления, но на самом деле он примерно
на 10% медленее -- или грубо говоря настолько же быстрый как и обычногое представление.
Это происходит поскольку дополнительные типы скрывают возможные оптимизации, которые программист
может сделать (и какие я сделал в случае обычного оптимизированного представления).

Вспомним тип `getMin_` в обычном предславлении и то, как он был использован функцией `getMin`.

~~~~haskell
getMin_ :: Ord a => Binom a -> (a, [Tree a], Binom a)

getMin :: Ord a => Binom a -> (a, Binom a)
getMin h = let (x,xs,h') = getMin_ h
           in (x,merge (list2binom xs Nil) h')
~~~~

Для оптимизированного представления он превращается в 

~~~~haskell
getMin_ :: Ord a => Binom a -> (a, Binom a -> Binom a, Binom a)

getMin :: Ord a => Binom a -> (a, Binom a)
getMin h = let (x,xs,h') = getMin_ h
           in (x,merge (xs Nil) h')
~~~~

Теперь рассмотрим оптимизированные вложенные типы. Мы не можем просто записать


~~~~~haskell
getMin_ :: Ord a => Binom_ a b -> (a, Binom_ a b -> Binom a, Binom_ a b)
~~~~~

поскольку мы не знаем ранг дерева во второй компоненте тройки. (Заметьте,
что если бы у нас были экзистенциальные типы, то мы могли бы записать

~~~~~haskell
getMin_ :: Ord a => Binom_ a b -> 
                    (a, exists c. Binom_ a c -> Binom a, Binom_ a b)

getMin :: Ord a => Binom a -> (a, Binom a)
getMin h = let (x,xs,h') = getMin_ h
           in (x,merge (xs Nil) h')
~~~~~

Некоторые реализации на самом деле поддерживают экзистенциальные типы,
но в ограниченных возможностях, и могут приводить к потере эффективности
при их использовании...)

Вместо возвращения дочерних элементов и затем применения их к `Nil`, 
мы можем сначала применить дочерние элементы к `Nil`, а затем вернуть
результат, получив тип
  
~~~~~~haskell
getMin_ :: Ord a => Binom_ a b -> (a, Binom a, Binom_ a b)

getMin :: Ord a => Binom a -> (a, Binom a)
getMin h = let (x,xs,h') = getMin_ h
           in (x,merge xs h')
~~~~~

Возможность записать функцию в таком виде опирается на ленивость, т.к. нам
требуется, чтобы вычислись только дочерние элементы настоящего минимума,
а не всех детей временных минимумов, которые мы получаем при построении.
Хотя это обозначает, что мы строим гораздо больше thunks вида `(xs Nil)`
при построении результата. И построение этих thunks очевидно достаточно
дорого.

Дополнение 2009 года:  Ralf Hinze предложил похожее представление в Разделе 6
"Numerical Representations as Higher-Order Nested Datatypes". Сегодня
вы скорее всего захотите использовать более хитрые структуры типов, такие
как зависимые типы или, например, Обобщенные алебраические типы, то все равно 
разве это не удивительно, насколько далеко мы можем зайти используя только
вложенные типы. 

