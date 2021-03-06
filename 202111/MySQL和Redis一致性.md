# MySQL 和 Redis 一致性

Redis作为缓存根据是否接收写请求可以分为两种情况，只读缓存和读写缓存。

对于只读缓存，只在缓存进行数据查找，即使用“更新数据库+删除缓存”策略；

对于读写缓存，需要在缓存中进行数据增删改查，即使用“更新数据库+更新缓存”策略。

## 针对只读缓存

只读缓存，数据库新增数据时，直接写入数据库；更新数据，需要删除缓存。后续访问时，发生缓存缺失，查询数据库，更新缓存。

对于新增数据时，写入数据库；访问数据时，缓存缺失，查询数据库，更新缓存，始终处于一致状态。

对于更新数据，有两种情况1.先更新数据库，再删除缓存；2.先删除缓存，再更新数据库。先删除缓存，再更新数据库可能会存在由于缓存缺失导致缓存穿透问题。建议优先选择先更新数据库，再删除缓存方案，另外，根据是否存在并发，主要有以下一致性问题：

1.没有并发，但是两个步骤一个成功、另一个失败。

2.有并发，其他线程读到数据旧值。

以下两两组合、分别讨论。

A. 没有并发，先更新数据库，再删除缓存。可能存在的情况是：数据库更新成功，缓存删除失败，导致后续缓存命中，读取到旧值。

B. 没有并发，先删除缓存，后更新数据库。可能存在的情况是：缓存删除成功，数据库更新失败，后续查询缓存缺失，查询数据库后读取到旧值。

针对上述两种情况可以采取的解决策略是：消息队列+异步重试。当操作第一步时，将第二步写入消息队列。如果第二步执行成功，则从消息队列中丢弃消息；如果失败，则重新读取消息进行重试。多次重试仍然失败的情况下，向业务层报错。

C. 有并发，先更新数据库，后删除缓存。A线程更新完数据库，尚未进行删除缓存时，B线程读取缓存命中，读取旧值。或者是A线程主库完成了更新，也完成了删除缓存，但是B线程读取缓存缺失时查询了从库，由于主从延时，从库尚未更新造成B线程读取旧值。

解决方案：延时删除可以减少由于主从延时导致的缓存旧值问题。可以通过加锁来保证一致性。

D. 有并发，先删除缓存，后更新数据库。A线程删除缓存成功，尚未更新数据库，B线程读取缓存缺失，查询数据库后更新缓存，然后A线程更新数据库造成数据不一致。

解决方案：缓存过期 + 延时双删：缓存过期可以保证虽然缓存中是旧值，但是过期之后从数据库中重新取到新值。延时双删，是指先删除缓存，后更新数据库，然后延时一段时间再删除一次缓存。使用延时双删可以进一步降低数据不一致的时间，存在的问题时延时时间不好估计。

## 针对读写缓存

读写缓存在缓存中进行数据增删改，并采取相应的回写策略，更新数据库。

同步直写：使用事务，保证更新缓存和更新数据库的原子性，并进行失败重试。

异步回写：操作时不更新数据库，等到数据淘汰时回写数据库。

一致性方面，同步直写>异步回写。如果需要保持强一致性，一般使用同步直写。

同步直写有两个操作，更新缓存和更新数据库，存在顺序先后问题，再结合并发读写，分以下情况讨论。

A.没有并发，先更新缓存，再更新数据库。存在更新数据库失败，导致不一致问题。

B.没有并发，先更新数据库，再更新缓存。存在更新缓存失败，导致不一致问题。

以上两种情况，均可以采用消息队列+异步重试解决方案，与前述类似，不再赘述。

C.有并发，先更新缓存，再更新数据库。A线程更新缓存成功，尚未更新数据库，B线程读取缓存命中，读取新值，虽然数据库和缓存存在不一致，但B仍读取新值，对业务影响小

D.有并发，先更新数据库，再更新缓存。A线程更新数据库成功，尚未更新缓存，B线程读取缓存命中，读取旧值。

解决方案：保存读取记录，延时比较，然后做业务补偿。

E.有并发，先更新缓存，再更新数据库。以下顺序A线程更新缓存，B线程更新缓存，B线程更新数据库，A线程更新数据库造成数据不一致。

F.有并发，先更新数据库，再更新缓存。以下顺序A线程更新数据库，B线程更新数据库，B线程更新缓存，A线程更新缓存造成数据不一致。

上述两种情况可以通过分布式锁解决。

## 小结

对于只读缓存，通常采用先更新数据库再删除缓存策略。消息机制+重试，延时消息（主从延时），加锁

对于读写缓存，通常采用先更新数据库再更新缓存策略。消息机制+重试，业务补偿，加锁

如需更强的一致性，可以通过分布式锁解决。

流程是：1.确定缓存类型（读写/只读）；2.确定一致性级别；3.确定同步/异步方式；4.确定缓存流程；5.补充细节。

## 其他问题

缓存穿透：空缓存、布隆过滤器

缓存击穿：过期时，对同一个Key的大量请求到数据库，互斥锁、永不过期

缓存雪崩：对多个Key的大量请求到数据库，随机失效时间，互斥锁

