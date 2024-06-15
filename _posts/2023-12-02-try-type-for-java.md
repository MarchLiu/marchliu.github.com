---
layout: post
title: "为 Java 编写的 Try 和 Tuple 类型"
description: "Java 传统上使用 try catch finally 关键字管理异常。这种机制久经考验。我参考 scala 的 Try 类型，编写了一组适用 java 的工具。"
category: tech
tags: ["java", "Jaskell", "try", "tuple"]
---

Java 设计了一套久经考验的异常处理机制，为开发高质量的程序提供了可靠保障。但是随着现在软件日趋复杂，特别是异步编
程的发展，语法级的异常处理机制已经不足使用。

我模仿 Scala 的 Try 类型，为 Java 编写了一组 Try 和 Tuple 方法，用于简化复杂应用逻辑中的异常管理和数据传递。
根据最新的 Java 21 和目前仍在广泛使用的 Java 8，分别在 jaskell-rocks 和  jaskell-java8 工具库中实现。

首先，这些工具并非内置异常语法的替代品，相反，它们依附于 try/catch 机制上，辅助我们更方便的使用异常管理。因此，
介绍它们的时候，也会涉异常机制的知识运用。


## 异常语法的局限

异常语法的限制往往不来自关键字本身，而是来自 Java 标准库。例如，stream api中通常只接受不抛出异常的函数对象，
但是在日常的操作中，异常是非常常见的，例如我们如果需要对一个 `List<String>` 中的每一个字符串做 json 解析，
不可避免的要面对可能出错的情况。那么 Stream 的 map 方法就很难满足要求。如果我们在 lambda 中明确的捕获异常，
这个代码块就会多出好几行。如果应用项目里经常要做类似的操作，那么这种高度重复的 try catch会遍地都是，而且难以
抽象。有些 JSON 库会吃掉这种错误直接返回 null。要我说这也不是很好的策略，至少不应该只有这么一种策略。我希望
将异常处理延后到某一个时刻，而不是在表达业务逻辑的代码中散落的到处都是。

与此类似的是，异步任务的各种抽象，例如各种 Future 类型，也需要无异常的代码块。在这些场景，能携带正常或异常数
据的 Try 封装，成为了一种可行的方案。

## Scala 的 for yield

首先，在介绍 Jaskell Try 类型之前，我们先看一下启发了这组设计的原型，Scala 强大的 for yield 语法：

````Scala
    for {
      a <- self(s)
      b <- p(s)
    } yield f(a, b))
````
这个例子来自 jaskell core 项目，它很简单，但是展示了 for yield 的重要特性。

首先，对于所有实现了 map 或 flatMap 方法，或者 iterator trait 的对象，可
以自动的经由 <- 运算符映射到左边的变量，然后这些变量能够被 yield 子句的代码
引用。这其中隐含的逻辑，相当于对 for 表达式中的各个引用项聚合为一个 tuple，
再使其传递到 yield 子句（其中还包含了 tuple 的隐式拆箱，或者类似 c sharp
的 var 类型的能力）。

当然我们在 java 的语法限制下，无法做到完美的复刻这种强大的能力，但是可以借助
一组 tuple，有限的模仿。

在这里讨论 for yield 和 tuple ，似乎有些跑题，但是这种 for yield 能力是
我非常关注的能力，它们为 Try 类型提供了实用价值，使其从我最初只是出于形式一
致性开始的基础工具设计实验，在多年后有了让我满意的实用性。

## Try 和 Triable

我们现在着重介绍一下 jaskell-rocks 项目中的 Try 类型。它充分利用了 java 
21 的新语法。而 jaskell-java8 的版本，基本上就是对应的朴素的 POJO 版本。

首先，我们定义 `Try<T>` 接口：

```Java
public sealed interface Try<T> permits Failure, Success {
    T get() throws Exception;

    boolean isOk();

    boolean isErr();
    // ...
}
```

这里仅给出了最基本的三个方法，其它工具方法我们在后续的内容中介绍。

从定义我们可以看到，Try 是一个封闭接口，它只有 Failure 和 Success 两个实现。下面是 Success 类型：

```Java
public record Success<T>(T item) implements Try<T> {
    @Override
    public T get() throws Exception {
        return item;
    }
        @Override
    public boolean isOk() {
        return true;
    }

    @Override
    public boolean isErr() {
        return false;
    }
    // ...
}
```

和 Failure 类型：

```Java
public record Failure<T>(Exception err) implements Try<T> {
    @Override
    public T get() throws Exception {
        throw err;
    }
        @Override
    public boolean isOk() {
        return false;
    }

    @Override
    public boolean isErr() {
        return true;
    }
    // ...
}
```

同样，我们暂且只给出最基础的三个方法。

这三个方法定义了 Try 的基本内核。它是可以携带异常状态的数据封装。因此，Success 和 Failure 对应的取值方法，分别只是
简单的返回包装值或者抛出包装的异常。对应的逻辑判定也是非常简单的固定逻辑。

注意观察，get 方法的声明提示了一个重要的问题，它必须声明为 throws Exception 。而我设计 Try 时，一个重要的动机就是
尽量少的显式处理异常，把异常处理集中在必要的位置。

所以，我们有必要设计一组方法，使我们可以不脱离 try 环境的前提下，执行正常的逻辑代码，允许它们抛出异常。

首先，我们定义一个不同于标准库的 `Supplier<T>` 抽象：

```Java
public interface Supplier<T> extends Triable<T> {
    T get() throws Exception;

    @Override
    default Try<T> collect() {
        try {
            return Try.success(get());
        } catch (Exception err) {
            return Try.failure(err);
        }
    }
}
```

这个抽象和 `java.util.function.Supplier<T>` 一样，也有一个 get 方法，但是它们的关键不同在
于，这个 get 带有 `throws Exception` 签名。

作为一个 SAM 类型，这个方法允许我们任意传递可能抛出异常的代码，因为实际使用时，我们尽量不直接调用
它，而是调用无异常抛出的 collect 方法，它总是返回已经求值的 Try 类型。

读者可能注意到，我们在这里还引用了一个称为 `Triable<T>` 的 interface：

```Java
public interface Triable<T> {
    Try<T> collect();
    // ...
}
```

和 Try 一样，Triable 也有大量的工具方法，但是它的核心只有这么一个抽象方法，其它都是基于它实
现（是的，都有 default 实现）的。

回到 Try，有了能够自动处理异常的 Supplier，我们可以为 Try 添加一个静态的构造方法：

```Java
    static <T> Try<T> tryIt(Supplier<? extends T> supplier) {
        try {
            return Try.success(supplier.get());
        } catch (Exception err) {
            return Try.failure(err);
        }
    }
```

于是，我们可以安全的使用可能抛出异常的代码作为lambda：

```Java
Try<Map<String, Object>> result = Try.tryit(() -> objectMapper.readValue(content));
```

这行代码简单的调用了 jackson 的 ObjectMapper 解析 json 字符串，转为对象字典。至于异常，那不是
现在需要担心的事。即使它失败了，那也合法的呆在 result 里呢，让下游代码去操心吧。

说到这个，我在写这篇文章的时候，才想来我还没有实现过 stream 方法。是的，我用 Try 已经很长时间了，
大规模改造它，并深度的在工作使用，也有了差不多三周，但是，处处不允许出现异常的 Stream，已经不那么
重要了。

只要有了类似 Supplier 的，可以管理异常的 Function，我完全可以自己实现一套对应的函数式抽象。于是，
我实现了 jaskell 版本的 `Function<T, U>`：

```Java
public interface Function<T, U> {
    U apply(T arg) throws Exception;

    default Try<U> collect(T arg) {
        try {
            return Try.success(apply(arg));
        } catch (Exception err) {
            return Try.failure(err);
        }
    }

    default <R> Function<T, R> andThen(Function<? super U, ? extends R> other) {
        return (T arg)-> other.apply(apply(arg));
    }

}
```

现在，我们可以任性的定义 map 和 flatMap：

```Java
    <U> Try<U> map(Function<T, U> mapper);

    <U> Try<U> flatMap(Function<? super T, Try<U>> mapper);
```

Success 的实现：

```Java
    @Override
    public <U> Try<U> map(Function<T, U> mapper) {
        try {
            return new Success<>(mapper.apply(item));
        } catch (Exception err) {
            return new Failure<>(err);
        }
    }
    
    @Override
    public <U> Try<U> flatMap(Function<? super T, Try<U>> mapper) {
        try {
            return mapper.apply(item);
        } catch (Exception e) {
            return Try.failure(e);
        }
    }
```

Failure 的版本比较简单，直接返回异常就可以：

```Java
    @Override
    public <U> Try<U> map(Function<T, U> mapper) {
        return new Failure<>(err);
    }
    
    @Override
    public <U> Try<U> flatMap(Function<? super T, Try<U>> mapper) {
        return new Failure<>(err);
    }
```

其它类似的方法，例如 filter 和 recover ，就不在这里讨论了，它们的实现都比较朴素。而下面介绍的这些功
能，对 Try 类型来说更为重要。

如前所述，for yield 语法一个非常重要的能力是将 monad 中的值 destruct 之后传递到 yield 中继续求值
和封装。这个工作在 jaskell 中，我是通过 try 类型的 join map/flatMap 和 tuple 类型实现的。

我们假设有这样一种场景，用户用纯文字的方式提交一个地址，我们的程序从中提取出省、市等信息（类似快递系统常见的
智能地址簿），这有可能成功，也有可能失败，而我们的下一步流程是需要将这些地址信息发送到当前环境中的
`Try<Order> post(String provinces, String city, String street, String house)` 。

```Java
class Address {
    private String origin;
    public Address(String origin) {
        this.origin = origin;
    }
    
    public Try<String> provinces() {
        // ...
    }
    
    public Try<String> city() {
        // ...
    }
    
    public Try<String> street() {
        // ...
    }
    
    public Try<String> house() {
        // ...
    }
}
```

那么，使用 Try 类型的 joinFlatMap 就是:

```Java
Try.joinFlatMap4(address.provinces(), address.city(), address.street(), address.house(),
        this::post);
```

如果其中某一个try求值失败，会直接返回一个 failure，只有四个参数都成功，才会执行 post。

如果我们需要将它们一起传递给下一步流程处理，那么可以使用 tuple 的 liftA 操作：

```Java
return Tuple4.liftA(address.provinces(), address.city(), address.street(), address.house())
```

如果传入的 Try 对象都成功，那么 liftA 返回 `Success<Tuple4<T, U, V, W>>` 类型对象，否则会返回第一个得
到的 failure。这符合我们使用 try catch 语法时的习惯，它就是在第一个异常发生时中断并抛出。但是从形式上，我
将异常处理和业务代码分离开，不必在每一个函数调用时就关注它的异常。

类似的，如果我们有一个 `Tuple4<String, String, String, String> tuple4`，也可以

```Java
tuple4.uncurry(this::post)
```

有时候，我们需要在 Try map/flatMap 中传递多个不同的 Triable 求值结果，Tuple 可以帮助我们很方
便的传递成组的信息。很奇怪的是，如此常用的功能，一直没有进入标准库，Apache Commons 有 Pair 和
TripleTuple ，有很多同行也实现了自己的 tuple 或 pair。我从我自己的工程经验出发，做了 tuple2
到 tuple8 共7个类型，目前看它们用作传递变量是足够了。

因为 Success 和 Failure 是两个 record，因此我们可以充分利用 java 21的模式匹配：

```Java
return switch(result){
    case Success(var result) -> {
        log.info("...");
        yield post(result)
    case Failure(var error) -> {
        log.error("...", error);
        rolback(....)
        yield ...
    }
}
```

## 注意事项

由于 Try 包装了 try catch 过程，如果我们可以忽略异常处理，程序也会正常运行，但这是有危险的，有可
能我们会因此错过发现问题的机会。我自己的业务代码中也遇到过这样的问题，在核查一个记账结果时，我发现
两份账目一直无法对齐，进一步检查发现某个返回 Try 的数据库写入一直失败，但是我没有用到那个 save
操作的返回值，也就没有发现它其实是 failure 。

Try 类型是用来简化异常捕获管理的，并不是用来替代 try catch，如果遇到用 try catch 更为有效的情况，
那么仍然应该使用 try catch。而 try 语法还兼顾资源自动回收和 finally 操作，这些因为作用域的关系，
在 Try 类型中实现抽象并不容易，目前还没有做到。