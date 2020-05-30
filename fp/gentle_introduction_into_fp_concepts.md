# Введение в промышленное функциональное программирование

TODO: Сделать введение

## Typeclass

### Обьектно-ориентированный подход к полиморфизму

Современное обьектно-ориентированное программирование в промышленной веб-разработке полагается на концепцию полиморфизма. Наследование считается антипаттерном и, если это возможно, не должно применяться. Это прямая трактовка принципа "Composition over inheritance" из "Design Patterns".
Реализацией этого принципа являются интерфейсы в языке Java, в которых до 8-ой версии не было возможности добавить реализацию методов.

Интерфейс определяет контракт:

```java
public interface Serializable {
    byte[] serialize();
}
```

Допустим у нас некий тип `DataRecord`, определенный вот так:
```java
public class DataRecord{
    public final String data;

    public DataRecord(String data) {
        this.data = data;
    }
}
```

Для того, чтобы реализовать контракт, a.k.a `interface`, необходимо использовать "квазинаследование" и реализовать метод
`serialize` внутри типа `DataRecord`. Выглядит это вот так:

```java
public class DataRecord implements Serializable {
    public final String data;

    public DataRecord(String data) {
        this.data = data;
    }

    @Override
    public byte[] serialize() {
        return data.getBytes();
    }
}
```

Т.е. для реализации поддержки всех необходимых контрактов для типа данных необходимо их все реализовать внутри определения этого типа данных.

Затем в коде вы можете использовать полиморфизм:

```java
public static void send(Serializable data) {
    byte[] bytes = data.serialize();
    //Реализуйте свою логику отправки байтов по сети
}

public static void main(String[] args) {
    send(new DataRecord("Мои данные"));
}
```

Де-факто это всё, что действительно нужно знать про ООП для промышленной разработки софта:
Не используйте наследование, используйте интерфейсы или аналогичные конструкции и композицию.


### Функциональный подход к полиморфизму

Функциональное программирование предлагает концепцию `typeclass` - класс типов. За страшным словом прячется простая идея: давайте разделять реализации контрактов для конкретных типов и определения этих типов. Как этого можно достичь? На Java это будет выглядеть так:

В определении интерфейса добавляется один generic параметр (параметр типа - type parameter в функциональных языках).
```java
public interface Serializable<A> {
    byte[] serialize(A value);
}
```
Далее, определим тип данных

```java
public class DataRecord{
    public final String data;

    public DataRecord(String data) {
        this.data = data;
    }
}
```

Реализация контракта для типа `DataRecord` будет выглядеть следующим образом
```java
public class SerializableForDataRecord implements Serializable<DataRecord> {
    @Override
    public byte[] serialize(DataRecord value) {
        return value.data.getBytes();
    }
}
```

Полиморфный код в этом случае будет выглядеть так: 
```java
public static <A> void send(A data, Serializable<A> serializer) {
    byte[] bytes = serializer.serialize(data);
    //Реализуйте свою логику отправки байтов по сети
}
```

Семантически эта конструкция означает следующее:
мы можем использовать метод `send` для любого типа `A`, для которого существует реализация интерфейса `Serializable<A>`. Таким образом проясняется название паттерна typeclass: `Serializable` определяет класс/набор/множество типов для которых реализован контракт `Serialiazable`.


Вот так выглядит использование полиморфного кода
```java
public static Serializable<DataRecord> serializableForDataRecord = new SerializableForDataRecord();

public static void main(String[] args) {
    send(new DataRecord("My data"), serializableForDataRecord);
}
```

Данная конструкция фактически имеет все те полезные свойства, что и классический полиморфизм с наследованием, однако мы имеем чёткое разделение контракта, типа данных и реализации контракта для этого типа данных.

Полученный код на Java выглядит чуть более многословно и это действительно так, что становится особенно видно при увелечении количества тайпклассов, используемых в коде. 

В Scala с имплиситами этот код выглядит следующим образом:

```Scala

trait Serializable[A]{
    def serialize(value: A): Array[Byte]
}
object Serializable{
    def apply[A: Serializable]: Serializable[A] = implicitly
}

case class DataRecord(data: String)

object DataRecord{
    implicit val serializable: Serializable[DataRecord] = new Serializable[DataRecord]{
        def serialize(value: DataRecord) = value.data.getBytes
    }
}

object Main extends App{
    def send[A: Serializable](value: A) = {
        val bytes = Serializable[A].serialize(value)
        //Ваш код по отправлению байтов
    }

    send(DataRecord("My data"))
}
```

Однако данный пример покрывает только тот случай, когда вся необходимая информация предоставлена во время компиляции. Мы вернёмся к вопросу динамического диспатчинга после обьяснения `ADT`.

### Algebraic Data Types или ADT и pattern matching

Функциональное программирование, как многие из нас уже наслышаны, нацелено в основном на преобразование данных. Но как именно определяются типы данных в функциональном мире?
Это очень просто, функциональщики говорят:
* Пусть у нас есть базовые типы данных, такие как `String`, `Char`, `Byte`, `Int`, `Double` и т.д.
* Операция умножения типов, которая работает следующим образом: `String` * `Byte` - это тип пары значений 
 типа `String` и `Byte`, т.е. буквально, при таком определении
 
 ```java
public class DataRecord{
    public final String data;
    public final byte number;

    public DataRecord(String data, Byte number) {
        this.data = data;
        this.number = number;
    }
}

 ``` 
 
  На Scala аналог выглядит так:  
  
  ```scala
  case class DataRecord(data: String, number: Byte)
  ```

 `DataRecord` является элементом типа `String * Byte`. Соответственно произведение трёх типов - это тип тройки значений и т.д.
 * Операция сложения типов, которая работает следующим образом: `String` + `Byte` - это либо значение типа `String`, либо значение типа `Byte`. В терминах наследования это выглядит так:
 
```java
 abstract class SumOfByteAndString {}
 final class ByteVersion extends SumOfByteAndString{
    public final byte number;
    
    public ByteVersion(byte number) {
        this.number = number;
    }
 }
 final class StringVersion extends SumOfByteAndString{
    public final String data;
    
    public ByteVersion(String data) {
        this.data = data;
    }
 }
```

Теперь с помощью `instanceof` мы можем понять в рантайме какая конкретно из альтернатив на пришла.
Но это не совсем полный аналог функционального сложения типов, потому что оно подразумевает, что других альтернатив быть не может, что не верно для приведённого кода на Java.

На Scala корректная версия произведения типов выглядит так:

```scala
sealed trait SumOfStringAndByte
case class StringVersion(data: String) extends SumOfStringAndByte
case class ByteVersion(data: Byte) extends SumOfStringAndByte
```

Ключевое слово `sealed` означает, что потомков `SumOfStringAndByte` можно определять только в файле, где он определён. Таким образом достигается свойство закрытости, которое не достигается на Java.

### Почему именно эти две операции?

Во-первых, с помощью этих двух операций можно определить все необходимые для переноса данных структуры.
Во-вторых, если мы знаем, что данные можно определять только таким образом, это значит, что мы знаем всё про их структуру, что активно используется при `pattern-matching`.

`Pattern-matching` работает следующим образом:

```scala
val sumValue: SumOfStringAndByte = ???
// ??? просто кидает исключение `NotImplemented`
sumValue match {
    case StringVersion(data) => ???
    case ByteVersion(data)   => ???
}
```

Благодаря тому, что мы знаем, что `SumOfStringAndByte` это сумма типов, язык позволяет выполнить исчерпывающий матчинг по возможным альтернативам и мы можем быть уверены, что матчинг будет корректный.

Для произведения типов это выглядит так:

```scala
val prodValue: DataRecord = ???

prodValue match{
    //Далее в этом case можно обращаться напрямую к значениям data и number
    case DataRecord(data: String, number: Byte) => ???
}

```

Здесь мы аналогичным образом смогли "заглянуть" внутрь типа, потому что мы знаем, что он был создан с помощью умножения типов.

Более того, паттерн матчинг позволяет "заглядывать в глубину", т.е. имея такой составной тип

```scala
 case class Inner(number: Int)
 case class Outer(str: String, inner: Inner)

 val outer: Outer = ???

 outer match {
     //Можем заглянуть внутрь Inner
     case Outer(str, Inner(number: Int)) => ???
 }
```

То же самое справедливо и для суммы типов:

```scala
sealed trait Outer
case class OuterLevel() extends Outer

sealed trait Inner extends Outer
case class InnerLevel() extends Inner
case class InnerLevelSecond() extends Inner

val outer: Outer = ???

outer match {
    //Мы видим альтернативы из всех уровней вложенности и исчерпываемость проверяется компилятором
    case OuterLevel()       => ???
    case InnerLevel()       => ???
    case InnerLevelSecond() => ???
}

//В тоже время имея только Inner мы всё ещё можем делать паттерн матчинг по нему
val inner: Inner = ???

inner match {
    case InnerLevel() => ???
    case InnerLevelSecond() = ???
}

```

Более того, паттерн матчинг работает, когда у нас используется и сложение и произведение типов:

```scala
sealed trait Sum
case class A() extends Sum
case class B() extends Sum

case class C(str: String, sum: Sum)


val c: C = ???

c match {
    //По альтернативе на каждый вариант Sum
    case (str, A()) => ???
    case (str, B()) => ???
}

sealed trait Sum2
case class C() extends Sum2
case class D() extends Sum2

case class TwoSums(sum1: Sum, sum2: Sum2)

val twoSums: TwoSums = ???

twoSums match {
    //По альтернативе на каждую возможную комбинацию
    case (A(), C()) => ???
    case (A(), D()) => ???
    case (B(), C()) => ???
    case (B(), D()) => ???
}
```

### Наследование

Главной фичей наследования является [динамический диспатчинг](https://en.wikipedia.org/wiki/Dynamic_dispatch) методов. Он же является и главным недостатком наследования. Динамический диспатчинг методов в наследовании перенасыщен. Множества [override](https://en.wikipedia.org/wiki/Method_overriding) и обращений к  `super()` только мешают читать код. Фактически, всё, что мы хотим от наследования - возможность диспатчить вызовы к потомкам. Это необходимый и достаточный минимум.

После обьяснения `ADT` легко заметить, что как раз таки сумма типов позволяет реализовать тот самый необходимый нам диспатчинг.

Допустим мы хотим сделать обобщённый контракт получения длины какой-то коллекции, на Java это будет выглядеть как-то так:

```java
interface Length{
    int length();
}

class MyListWithLength<A> implements Length{
    final List<A> list;

    public <A> MyListWithLength(List<A> list){
        this.list = list;
    }

    @Override
    int length(){
      return list.length();   
    }
}

class MyStringWithLength implements Length{
    final String str;

    public MyStringWithLength(String str){
        this.str = str;
    }
    
    @Override
    int length(){
      return str.length();   
    }
}

//

public static Length someParsedElement = throw new Exception("Not implemented")

public static void printLength(Length data) {
    System.out.println("Length" + data.length());
    //Реализуйте свою логику отправки байтов по сети
}

public static void main(String[] args) {
    //В рантайме вызовет нужный метод
    printLength(someParsedElement);
    printLength(new MyStringWithLength("Мои данные"));
    printLength(new MyListWithLength(Arrays.asList(1, 2, 3)));
}
```

На Scala это выглядит следующим образом:

```scala
trait Length[A]{
    def length(value: A)
}

sealed trait StringOrIntList
case class StringWrapper(str: String) extends StringOrIntList
case class IntListWrapper(list: List[Int]) extends StringOrIntList

object Length{
    implicit val stringInstance: Length[StringWrapper] = new Length[StringWrapper]{
        def length(value: StringWrapper) = value.str.length
    }
    implicit val listInstance: Length[IntListWrapper] = new Length[IntListWrapper]{
        def length(value: IntListWrapper) = value.list.length
    }  
    //Инстанс для диспатчинга
    //(бойлерплейт от которого можно и нужно избавиться макросами, приведён без них для простоты)
    implicit val stringOrIntListInstance: Length[StringOrIntList] = new Length[StringOrIntList]{
        def length(value: StringOrIntList) = value match {
            case wrapper@IntListWrapper(_)  => Length[IntListWrapper].length(wrapper)
            case wrapper@StringWrapper(_)   => Length[StringWrapper].length(wrapper)
        }
    }  
}


object Main extends App{
    def parse: StringOrIntList = ???

    def printLength[A: Length](value: A) = println(s"Length: ${Length[A].length(value)}")

    //То же самое, что printLength[StringOrIntList](value)(stringOrIntListInstance)
    printLength(parse)
}

```

Таким образом мы получили диспатчинг без наследования, используя `ADT` и `typeclasses`.
