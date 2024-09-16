# Трансформация Bot API

Middleware --- это функция, которая обрабатывает объект контекста, т.е. входящие данные.

grammY также предоставляет вам противоположность этому.
Функция _трансформатор_ --- это функция, которая обрабатывает исходящие данные, т.е.

- название метода API бота, который необходимо вызвать, и
- объект `payload`, соответствующий методу

Вместо того чтобы иметь `next` в качестве последнего аргумента для вызова нижележащего middleware, вы получаете `prev` в качестве первого аргумента для использования функций вышележащего трансформатора.
Взглянув на сигнатуру типа `Transformer` ([ссылка на grammY API](/ref/core/transformer)), мы можем увидеть, как она отражает это.
Обратите внимание, что `Payload<M, R>` ссылается на `payload`, который должен соответствовать данному методу, а `ApiResponse<ApiCallResult<M, R>> --- это тип возврата вызванного метода.

Последняя вызываемая трансформирующая функция --- это встроенный вызывающий элемент, который выполняет такие действия, как сериализация определенных полей JSON и, в конечном счете, вызов `fetch`.

Нет эквивалента класса `Composer` для трансформирующей функций, потому что это, вероятно, излишне, но если вам это нужно, вы можете написать свой собственный.
PR приветствуется! :wink:

## Установка трансформирующей функции

Трансформирующие функции могут быть установлены в `bot.api`.
Вот пример трансформирующей функции, которая ничего не делает:

```ts
// Трансформирующая функция, которая ничего не делает
bot.api.config.use((prev, method, payload, signal) =>
  prev(method, payload, signal)
);

// Сравнение с таким же middleware
bot.use((ctx, next) => next());
```

Вот пример трансформирующей функции, которая предотвращает все вызовы API:

```ts
// Некорректно возвращают undefined вместо соответствующих типов объектов.
bot.api.config.use((prev, method, payload) => undefined as any);
```

Вы также можете установить трансформирующие функции в API-объект контекстного объекта.
Тогда трансформирующая функциия будет временно использоваться только для API-запросов, которые выполняются для этого конкретного контекстного объекта.
Вызовы к `bot.api` остаются незатронутыми.
Вызовы через контекстные объекты параллельно запущенного middleware также остаются незатронутыми.
Как только соответствующий middleware завершает свою работу, трансформирующая функция отбрасывается.

```ts
bot.on("message", async (ctx) => {
  // Устанавливается на все контекстные объекты, обрабатывающие сообщения.
  ctx.api.config.use((prev, method, payload, signal) =>
    prev(method, payload, signal)
  );
});
```

> Параметр `signal` должен всегда передаваться в `prev`.
> Он позволяет отменять запросы и важен для работы `bot.stop`.

Трансформирующие функции, установленные на `bot.api`, будут предустановлены в каждый объект `ctx.api`.
Таким образом, вызовы к `ctx.api` будут преобразованы как теми трансформаторами, которые находятся в `ctx.api`, так и теми трансформаторами, которые установлены в `bot.api`.

## Случаи использования трансформирующих функций

Трансформирующие функции так же гибки, как и middleware, и имеют столько же различных применений.

Например, плагин [grammY menu](../plugins/menu) устанавливает трансформирующую функцию для преобразования исходящих экземпляров меню в правильный payload.
Вы также можете использовать их для

- реализации [контроля флуда](../plugins/transformer-throttler),
- имитировать API-запросы во время тестирования,
- добавить [повторение запросов](../plugins/auto-retry), или
- многое другое.

Обратите внимание, что повторный вызов API может иметь странные побочные эффекты: если вы вызовете `endDocument` и передадите экземпляр потока, пригодного для чтения, в `InputFile`, то поток будет прочитан при первой попытке запроса.
Если вы снова вызовете `prev`, поток может быть уже (частично) использован, что приведет к битым файлам.
Поэтому более надежным способом является передача путей к файлам в `InputFile`, чтобы grammY мог воссоздать поток по мере необходимости.

## Расширитель API

В grammY есть [расширители контекста](../guide/context#расширители-контекста), которые можно использовать для настройки типа контекста.
Сюда входят методы API как те, которые находятся непосредственно в объекте контекста, такие как `ctx.reply`, так и все методы в `ctx.api` и `ctx.api.raw`.
Однако вы не можете изменять типы `bot.api` и `bot.api.raw` с помощью контекстных расширителей.

Именно поэтому grammY поддерживает _Расширители API_.
Они решают эту проблему:

```ts
import { Api, Bot, Context } from "grammy";
import { SomeApiFlavor, SomeContextFlavor, somePlugin } from "some-plugin";

// Расширитель контекста
type MyContext = Context & SomeContextFlavor;
// Расширитель API
type MyApi = Api & SomeApiFlavor;

// Использование двух расширителей
const bot = new Bot<MyContext, MyApi>("");

// Использование плагина
bot.api.config.use(somePlugin());

// Теперь вызовите `bot.api` с настроенными типами из расширителя API.
bot.api.somePluginMethod();

// Кроме того, используйте настроенный тип контекста из расширителя котекста.
bot.on("message", (ctx) => ctx.api.somePluginMethod());
```

Расширители API работают точно так же, как и контекстные расширители.
Существуют как аддитивные, так и трансформируемые расширители API, и несколько расширителей API можно комбинировать так же, как это делается с контекстными расширителями.
Если вы не знаете, как это работает, вернитесь к [разделу о контекстных расширителях](../guide/context#расширители-контекста) в руководстве.