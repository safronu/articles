# Введение в промышленное функциональное программирование

## Typeclass

### Обьектно-ориентированный подход к полиморфизму

Современное обьектно-ориентированное программирование в промышленной веб разработке полагается на концепцию полиморфизма. Наследование считается антипаттерном и не должно быть использовано, если это возможно. Это прямая трактовка принципа "Composition over inheritance" из "Design Patterns".
Реализацией этого принципа являются интерфейсы в языке Java, в которых до 8ой версии не было возможность добавить реализацию методов.

Интерфейс определяет контракт:

```java
public interface Serializable {
    byte[] serialize();
}
```

Теперь, чтобы для какого-то типа, например такого
```java
public class DataRecord{
    public final String data;

    public DataRecord(String data) {
        this.data = data;
    }
}
```

Реализовать контракт a.k.a `interface` необходимо использовать "квазинаследование" и реализовать метод
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

Т.е. реализация поддержки всех необходимых контрактов для типа данных необходимо их все реализовать внутри этого типа данных.

Затем в коде вы можете использовать полиморфизм:

```java
public static void send(Serializable data) {
    byte[] bytes = data.serialize();
    //Реализуйте свою логику отправления байтов по сети
}

public static void main(String[] args) {
    send(new DataRecord("Мои данные"));
}
```

Де-факто это всё, что действительно нужно знать про ООП для промышленной разработки софта:
Не используй наследование, используй интерфейсы(или аналогичные конструкции) и композицию.


### Функциональный подход к полиморфизму

Функциональное программирование предлагает концепцию `typeclass`(или класс типов). За страшным словом прячется простая идея: давайте разделять реализации контрактов для конкретных типов и определения этих типов. Как этого можно достичь? На Java это будет выглядеть так:

В определении интерфейса добавляться один generic параметр(или параметр типа a.k.a type parameter в функциональных языках).
```java
public interface Serializable<A> {
    byte[] serialize(A value);
}
```
Далее, имея такой тип данных

```java
public class DataRecord{
    public final String data;

    public DataRecord(String data) {
        this.data = data;
    }
}
```

Реализация контракта будет выглядеть следующим образом
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
    //Реализуйте свою логику отправления байтов по сети
}
```
Семантически эта конструкция означает следующее:
мы можем использовать метод `send` для любого типа `A`, для которого существует реализация интерфейса `Serializable<A>`. Таким образом проясняется название паттерна typeclass. Так как `Serializable` определяет класс/набор/множество типов для которых реализован контракт `Serialiazable`


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