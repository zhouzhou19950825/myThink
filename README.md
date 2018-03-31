# 简单描述Java8-Stream中ForEach的运作原理

由于好久好久没写文章了，也没啥时间，今天周末抽个时间写一篇关于Java8中的Stream的foreach描述

- **Stream中ForEach的基本用法**
-  **可分迭代器 Spliterator<T>**
- **ReferencePipline , ReferencePipline.Head**
- **Stream中可设置isParallel（）和sequential（），在具体运行中如何被使用**


## Stream中ForEach的基本用法

> 接口

```
void forEach(Consumer<? super T> action);
```

使用1：

```
Stream.generate(random).limit(10).forEach(System.out::println);//可传入方法
```
使用2：

```
roster.stream().parallel().filter(p1.negate()).forEach(p -> t.test(p));//也可以实现接口
```
>这里引入了.parallel() 这是并行运行的标志，还有个sequential() 顺序执行，在归约操作之前，这两者可以切换使用，但是要慎重。Parallel()的使用考虑到流长度、分割最小单位等考虑，他并不是肯定比顺序流遍历快，而是看流的量，如果用到limit这些操作，不建议使用并行流，物极必反，并行流的代价就是消耗cpu，这中间的时间不可估计。


##可分迭代器Spliterator<T>

他有几个接口方法

```
boolean tryAdvance(Consumer<? super T> action);
Spliterator<T> trySplit();
long estimateSize();
int characteristics();

```
>与往常一样，T是Spliterator遍历的元素的类型。tryAdvance方法的行为类似于普通的 Iterator，因为它会按顺序一个一个使用Spliterator中的元素，并且如果还有其他元素要遍历就返回true。但trySplit是专为Spliterator接口设计的，因为它可以把一些元素划出去分给第二个Spliterator（由该方法返回），让它们两个并行处理。Spliterator还可通过 estimateSize方法估计还剩下多少元素要遍历，因为即使不那么确切，能快速算出来是一个值也有助于让拆分均匀一点。重要的是，要了解这个拆分过程在内部是如何执行的，以便在需要时能够掌控它。

实现类：
![这里写图片描述](http://img.blog.csdn.net/20180331091731192?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzg3MjQyOTU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

>这是他的实现类（部分），所以为什么LongStream、IntegerStream、DoubleStream、List等能直接forEach并可以并行遍历操作归功于Spliterator.

##ReferencePipline与ReferencePipline.Head

>ReferencePipeline他是一个抽象类，继承AbstractPipeline和实现Stream类，可以理解为一个遍历的管道的抽象类。
ReferencePipeline有一个静态内部类Head，并且也继承了ReferencePipeline

我们不妨看看Heap相关的方法，首先我们看下他的构造器

```
Head(Spliterator<?> source,
             int sourceFlags, boolean parallel) {
            super(source, sourceFlags, parallel);
        }

```
>source:可分迭代器
parallel:是否并行操作
sourceFlags:输入源管道元素类型标志（this.sourceOrOpFlags = sourceFlags & StreamOpFlag.STREAM_MASK;）

>forEach方法的实现

```
       @Override
        public void forEach(Consumer<? super E_OUT> action) {
            if (!isParallel()) {
                sourceStageSpliterator().forEachRemaining(action);
            }
            else {
                super.forEach(action);
            }
        }

```
>首先判断是否使用并行流，如果不是就是普通的forEach遍历，也就是通过forEachRemaining方法(顺序流)。
>
如果是并行流：
    通过接口TerminalOp调用evaluateParallel方法，有四种实现REFERENCE, INT_VALUE, LONG_VALUE, DOUBLE_VALUE

看下ForEachOps中有个ForEachOp抽象类实现

```
		@Override
        public <S> Void evaluateParallel(PipelineHelper<T> helper,
                                         Spliterator<S> spliterator) {
            if (ordered)
                new ForEachOrderedTask<>(helper, spliterator, this).invoke();
            else
                new ForEachTask<>(helper, spliterator, helper.wrapSink(this)).invoke();
            return null;
        }
```

>看到这里的时候已经很显然易见了，Stram分装的foeach最终的运行是用过jdk7引进fork/join框架。
由于这篇文章主要描述关于jdk8中Steam中foreach的原理，下次抽个时间再写一篇关于fork/join的运行原理。

##Stream中可设置isParallel（）和sequential（）
>Stream中的顺序遍历和并行遍历是可控的，比如说map操作是并行的，但是filter操作是顺序的，这个都可以的。还是一句话，不要盲目乱使用。

```
if (!isParallel()) {
                sourceStageSpliterator().forEachRemaining(action);
            }
            else {
                super.forEach(action);
            }
```
>每次遍历都会判断当前操作是并行还是串行。




今天先写到这里，找个时间再详细写写，把一些个人所觉得有意思的代码程序分享出来
