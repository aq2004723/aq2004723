# java full gc 频繁
排查full gc问题首先要知道什么情况下会发生full gc


> 作者：Ted Mosby  
> 链接：https://www.zhihu.com/question/41922036/answer/144566789  
> 来源：知乎  
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。  
> 1. Full GC定义是相对明确的，就是针对整个新生代、老生代、元空间（metaspace，java8以上版本取代perm gen）的全局范围的GC；  
>   
> 2. Minor GC和Major GC是俗称，在Hotspot JVM实现的Serial GC, Parallel GC, CMS, G1 GC中大致可以对应到某个Young GC和Old GC算法组合；  
>   
> 3. 最重要是搞明白上述Hotspot JVM实现中几种GC算法组合到底包含了什么。  
> 3.1 Serial GC算法：Serial Young GC ＋ Serial Old GC （敲黑板！敲黑板！敲黑板！实际上它是全局范围的Full GC）；  
> 3.2 Parallel GC算法：Parallel Young GC ＋ 非并行的PS MarkSweep GC / 并行的Parallel Old GC（敲黑板！敲黑板！敲黑板！这俩实际上也是全局范围的Full GC），选PS MarkSweep GC 还是 Parallel Old GC 由参数UseParallelOldGC来控制；  
> 3.3 CMS算法：ParNew（Young）GC + CMS（Old）GC （piggyback on ParNew的结果／老生代存活下来的object只做记录，不做compaction）＋ Full GC for CMS算法（应对核心的CMS GC某些时候的不赶趟，开销很大）；3.4 G1 GC：Young GC + mixed GC（新生代，再加上部分老生代）＋ Full GC for G1 GC算法（应对G1 GC算法某些时候的不赶趟，开销很大）；  
>   
> 4. 搞清楚了上面这些组合，我们再来看看各类GC算法的触发条件。简单说，触发条件就是某GC算法对应区域满了，或是预测快满了。比如，  
> 4.1 各种Young GC的触发原因都是eden区满了；  
> 4.2 Serial Old GC／PS MarkSweep GC／Parallel Old GC的触发则是在要执行Young GC时候预测其promote的object的总size超过老生代剩余size；  
> 4.3 CMS GC的initial marking的触发条件是老生代使用比率超过某值；  
> 4.4 G1 GC的initial marking的触发条件是Heap使用比率超过某值，跟4.3 heuristics 类似；  
> 4.5 Full GC for CMS算法和Full GC for G1 GC算法的触发原因很明显，就是4.3 和 4.4 的fancy算法不赶趟了，只能全局范围大搞一次GC了（相信我，这很慢！这很慢！这很慢！）；  
> 5 题主说的 “Full GC会先触发一次Minor GC” － 指的应该是5.1 （说错了，我删了）  
> 5.2 PS MarkSweep GC／Parallel Old GC（Full GC）之前会跑一次Parallel Young GC；原因就是减轻Full GC 的负担。哇～整个picture 是有点乱，希望我整理的还算清楚：）  

总结起来，full gc的触发根据回收器的不同有以下几种可能

1. 老年代空间不足
2. Permanet Generation空间满
3. CMS GC时出现promotion failed和concurrent mode failure
4. Serial Old GC／PS MarkSweep GC／Parallel Old GC的触发则是在要执行Young GC时候预测其promote的object的总size超过老生代剩余size


对应的表现以及解决方式：
## 老年代空间不足
连续多次触发full gc，old gen 空间占比高，甚至直接抛出`java.lang.OutOfMemoryError: Java heap space `

解决方案，调整jvm参数，扩大老年代比率，或者直接扩大堆空间

## Permanet Generation空间满

连续多次触发full gc， Permanet Generation空间占比高

解决方案：扩大Perm 大小

## CMS GC时出现promotion failed和concurrent mode failure
查询gc日志
promotion failed是在进行Minor GC时，survivor space放不下、对象只能放入旧生代，而此时旧生代也放不下造成的；concurrent mode failure是在执行CMS GC的过程中同时有对象要放入旧生代，而此时旧生代空间不足造成的。

解决方案：增大survivor space、old gen 空间

## Serial Old GC／PS MarkSweep GC／Parallel Old GC的触发则是在要执行Young GC时候预测其promote的object的总size超过老生代剩余size
解决方案：增大old gen 空间












