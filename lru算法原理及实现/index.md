# LRU算法原理及实现


# LRU算法原理及实现

## 前言

> 什么是LRU算法？

LRU（Least recently used，最近最少使用）算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。

那么该数据结构就是当存储队列到达上限时，清除的是最久未被访问的节点，该节点一般认为是最可能无用的节点，保留下来的是最近都有使用过的节点，因此可以实现对"有用"数据的最大程度保留。

![img](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/v2-71b21233c615b1ce899cd4bd3122cbab_720w.jpg)

> LRU算法应用场景？

LRU算法有许多的应用场景。

1. Redis中使用LRU来进行淘汰
2. 操作系统底层的内存管理，比如说页面置换算法中的LRU算法
3. 业务处理，比如说做一个用户最近10个浏览记录，那么就可以使用LRU算法来维护一个大小为10的LRU队列

---

## 原理

> LRU算法需要实现如下特性

1. 实现get/put方法(都为O(1)的时间复杂度)
2. 每次get时需要将访问的节点提前至队首
3. 每次put需要判断队列是否已满，满了则将最后的节点删除，并且将该节点放至队首，不满则直接放队首

> 基于上述特性需要实现如下数据结构

1. 首先需要实现队列，如果使用单向链表，当我们需要使用删除操作时，需要获得前置节点的指针，单向链表则不能做到直接获取。因此使用双向链表。
2. 又我们需要get方法达到O(1)的时间复杂度，因此需要一个Hashmap，可以根据key定位到我们双向链表的Node节点。
3. 由于我们HashMap中有key，所以我们可不可以Node中只存value，其实是不可以的，后续会提到这个原因。

**因此我们实现了如下数据结构**

![HashLinkedList](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/4.jpg "双向链表+HashMap的数据结构")

**缓存淘汰过程如下**

![LRU算法缓存淘汰策略- Mr.Ming2 - 博客园](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/407408-20180321102219351-2030402661.png "LRU缓存淘汰过程")

## 实现

那么现在我们就开始实现一个简易的LRU算法

> 首先需要实现一个Node节点

```java
public class Node {

    //Node中存键值对
    public int key,val;

    //前置节点和后置节点
    public Node next,pre;

    public Node (int key,int val){
        this.key = key;
        this.val = val;
    }
}
```

> 接下来实现一个关于Node节点的双向链表以及相关方法

1. 需要有头尾节点
2. 有大小size
3. 实现addFirst()/removeLast()/remove()/size()方法

```java
/**
 * @author TheR1sing3un
 * @date 2021/12/30 11:14
 * @description 双向链表数据结构实现
 */

public class DoubleList {

    //头节点和尾节点
    private Node head,tail;

    //大小
    private int size;

    /**
     * 构造方法并且完成首尾节点初始化
     */
    public DoubleList (){
        head = new Node(0,0);
        tail = new Node(0,0);
        head.next = tail;
        tail.pre = head;
    }

    /**
     * 添加一个节点到队首
     * @param node
     */
    public void addFirst(Node node){
        node.pre = head;
        node.next = head.next;
        head.next.pre = node;
        head.next = node;
        size++;
    }

    /**
     * 删除最后一个节点
     * @return 返回被删除的节点
     */
    public Node removeLast(){
        //没有节点时候,返回null
        if (head.next == tail) return null;
        Node last = tail.pre;
        last.pre.next = tail;
        tail.pre = last.pre;
        size--;
        return last;
    }

    /**
     * 删除某个节点
     * @param node
     * @return 返回被删除的节点
     */
    public Node remove(Node node){
        node.pre.next = node.next;
        node.next.pre = node.pre;
        size--;
        return node;
    }

    /**
     * 获取当前链表大小
     * @return 链表大小
     */
    public int size(){
        return size;
    }

    /**
     * 打印该链表
     */
    public void print(){
        Node cur = head.next;
        while(cur != tail){
            System.out.print(cur.key+"->"+ cur.val+" ");
            cur = cur.next;
        }
        System.out.println();
    }

}
```

> 接下来实现LRU队列

1. 需要有一个HashMap完成key->Node的映射
2. 需要有一个DoubleList来存放Node节点
3. 有容量上限cap
4. 使用私有的delete()/removeLast()/makeFirst()/addFirst()来辅助put()和get()方法，避免直接操作node
5. 有put()/get()/size()方法

```java
/**
 * @author TheR1sing3un
 * @date 2021/12/30 11:49
 * @description LRU队列实现
 */

public class LRUCache {

    //key->node的映射
    private HashMap<Integer,Node> map;

    //双向链表作为缓存队列
    private DoubleList cache;

    //最大容量
    private int cap;

    //构造方法,并且完成map和list的初始化
    public LRUCache(int cap){
        this.cap = cap;
        map = new HashMap<Integer,Node >();
        cache = new DoubleList();
    }

    /**
     * 删除某个key对于的键值对
     * @param key
     */
    private void delete(int key){
        //从map中获取该节点
        Node node = map.get(key);
        //从链表删除该节点
        cache.remove(node);
        //从map中删除该key
        map.remove(key);
    }

    /**
     * 删除缓存队列中最后一个节点,也就是最久未被使用的节点
     */
    private void removeLast(){
        //从链表中删除该节点
        Node node = cache.removeLast();
        //根据该节点的key来删除map中对应的键值对
        map.remove(node.key);
        //此处也就是为什么我们需要Node中存key的原因,因为需要根据key来删除map中的键值对
    }

    /**
     * 将某个键提前到队首,也就是变成最近使用的节点
     * @param key
     */
    private void makeFirst(int key){
        //从map中获取该节点
        Node node = map.get(key);
        //删除该节点
        cache.remove(node);
        //将该节点重新插入到队首
        cache.addFirst(node);
    }

    /**
     * 添加一个键值对到队首,也就是到最近使用的位置
     * @param key
     * @param val
     */
    private void addFirst(int key,int val){
        Node node = new Node(key,val);
        //添加到队首
        cache.addFirst(node);
        //添加映射
        map.put(key,node);
    }

    /**
     * 暴露出来的put()方法,向LRUCache中插入一个键值对,若key已存在则更新
     * @param key
     * @param val
     */
    public void put(int key,int val){
        if (map.containsKey(key)){
            //若key已存在,那么需要先删除旧数据
            delete(key);
            //插入到队首
            addFirst(key,val);
            return;
        }
        //当达到容量上限时
        if(cache.size() == cap){
            //删除最久未被使用的节点
            removeLast();
        }
        //添加到队首
        addFirst(key,val);
    }

    /**
     * 暴露出来的get()方法,根据key获取到val,若不存在,则返回-1(假定val都为正整数)
     * @param key
     */
    public int get(int key){
        if (!map.containsKey(key)){
            //key不存在
            return -1;
        }
        //将该key移动到队首,也就是最近使用的第一个
        makeFirst(key);
        //返回值　
        return map.get(key).val;
    }

    /**
     * 暴露出来的size()方法,返回当前的大小
     * @return 返回当前队列大小
     */
    public int size(){
        return cache.size();
    }

    /**
     * 打印缓存队列
     */
    public void print(){
        cache.print();
    }
}
```

---

## 测试

```java
public class Test {
    public static void main(String[] args) {
        LRUCache lruCache = new LRUCache(5);
        Scanner sc = new Scanner(System.in);
        int key,val;
        while(true){
            System.out.println("请输入插入的key和val(以空格隔开,输入-1则结束)");
            key = sc.nextInt();
            if (key == -1) break;
            val = sc.nextInt();
            lruCache.put(key,val);
            lruCache.print();
        }
        while(true){
            System.out.println("请输入需要获取的key(输入-1则结束)");
            key = sc.nextInt();
            if (key == -1) break;
            System.out.println("key->val: "+key+"->"+lruCache.get(key));
            lruCache.print();
        }
    }
}
```

![image-20211230132907832](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211230132907832.png "自行输入测试")

**至此，我们就完成了一个简易的LRU算法，除此之外，我们还可以使用Java中自带的api来简化LRU实现**

---

## LRU(使用LinkedHashMap)

LinkedHashMap内部数据结构就是一条双向链表+HashMap，因此我们可以不用自己定义这些数据结构，具体实现交给LinkedHashMap，我们只需要处理put()/get()方法即可。

```java
/**
 * @author TheR1sing3un
 * @date 2021/12/30 13:35
 * @description 使用LinkedHashMap实现的LRU
 */

public class LRUCacheSimple {

    private int cap;

    private LinkedHashMap<Integer,Integer> cache = new LinkedHashMap();

    public LRUCacheSimple(int cap){
        this.cap = cap;
    }

    /**
     * 将某个键改成最近使用的
     * @param key
     */
    private void makeRecently(int key){
        //先获取值
        Integer val = cache.get(key);
        //删除该键
        cache.remove(key);
        //重新插入
        cache.put(key,val);
    }

    /**
     * 插入key,val
     * @param key
     * @param val
     */
    public void put(int key,int val){
        if (cache.containsKey(key)){
            //如果已存在,那么覆盖
            cache.put(key,val);
            //将key变成最近使用
            makeRecently(key);
            return;
        }
        if (cap <= cache.size()){
            //当满了之后
            //获取第一个key(也就是LinkedHashMap中最久未被访问的节点)
            Integer first = cache.keySet().iterator().next();
            //删除该节点
            cache.remove(first);
        }
        //将新的key,val插入
        cache.put(key,val);
    }

    /**
     * 根据key获取value(假定value都是正整数)
     * @param key
     * @return value(-1表示不存在)
     */
    public int get(int key){
        if (!cache.containsKey(key)){
            return -1;
        }
        //将该key变成最近使用的
        makeRecently(key);
        return cache.get(key);
    }

    /**
     * 打印该缓存
     */
    public void print(){
        Iterator<Integer> iterator = cache.keySet().iterator();
        while(iterator.hasNext()){
            int key = iterator.next();
            System.out.print(key+"->"+ cache.get(key)+" ");
        }
        System.out.println();
    }
}
```

---

## LRU(GoLang实现)

上述是Java版本的LRU实现，下述我使用Golang语言进行实现，大致思路和逻辑是相同的。

> 代码如下

```go
package LRU

import "fmt"

type Node struct {
   key  int
   val  int
   pre  *Node
   next *Node
}

type LRUCache struct {
   Cap    int           //最大容量
   bucket map[int]*Node //HashMap
   head   *Node
   tail   *Node
   Size   int
}

//LRUCache的构造
func New(cap int) *LRUCache {
   cache := &LRUCache{
      Cap:    cap,
      bucket: make(map[int]*Node, cap),
      head:   &Node{0, 0, nil, nil},
      tail:   &Node{0, 0, nil, nil},
      Size:   0,
   }
   cache.head.next = cache.tail
   cache.tail.pre = cache.head
   return cache
}

//添加一个节点到首位(也就是最近一个访问的)
func (this *LRUCache) addNodeFirst(node *Node) {
   //判断是否到容量上限
   if this.Size == this.Cap {
      //到达上限之后,删除最后一个节点
      this.deleteNode(this.tail.pre)
   }
   //添加节点到首位
   node.pre = this.head
   node.next = this.head.next
   this.head.next.pre = node
   this.head.next = node
   //添加该映射
   this.bucket[node.key] = node
   this.Size++
}

//将某个key变成最近使用的(假定该key一定存在)
func (this *LRUCache) makeNodeFirst(key int) {
   //根据key获取该节点
   node := this.bucket[key]
   //先删除该节点
   this.deleteNode(node)
   //再加入该节点到首位
   this.addNodeFirst(node)
}

//删除某个节点
func (this *LRUCache) deleteNode(node *Node) {
   //先删除映射
   delete(this.bucket, node.key)
   //从双向链表中删除该节点
   node.pre.next = node.next
   node.next.pre = node.pre
   this.Size--
}

//put方法
func (this *LRUCache) Put(key, val int) {
   //先判断是否已有该节点,有则更新
   node := this.bucket[key]
   if node == nil {
      //当没有该节点时,直接加入到首位
      node := &Node{key, val, nil, nil}
      this.addNodeFirst(node)
   } else {
      //如果已经有该节点,那么先直接更新该节点的值并且提前至首位
      node.val = val
      this.makeNodeFirst(key)
   }
}

//get方法,如果不存在则返回-1(假设值都是正数)
func (this *LRUCache) Get(key int) int {
   node := this.bucket[key]
   if node == nil {
      //不存在则返回-1
      return -1
   } else {
      //将节点提前到首位
      this.makeNodeFirst(key)
      //返回值
      return node.val
   }
}

func (this *LRUCache) Print() {
   cur := this.head.next
   for ; cur != this.tail; cur = cur.next {
      fmt.Printf("%d->%d ", cur.key, cur.val)
   }
   fmt.Println()
}
```

> 测试代码如下

```go
package LRU

import (
   "fmt"
   "testing"
)

func TestLRUCache_New(t *testing.T) {
   cache := New(5)
   for i := 1; i < 8; i++ {
      fmt.Printf("加入节点%d\n", i)
      cache.Put(i, 11*i)
      cache.Print()
   }
   fmt.Printf("查询节点%d ,值为%d\n", 3, cache.Get(3))
   cache.Print()
   fmt.Printf("查询节点%d ,值为%d\n", 6, cache.Get(6))
   cache.Print()
   fmt.Printf("查询节点%d ,值为%d\n", 7, cache.Get(7))
   cache.Print()
}
```

![image-20220101151943479](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220101151943479.png)

---

## 优化

上述的LRU容器还是一个根本不能投入生产使用的玩具级实现，可以在进一步进行优化。

> 值的类型

上述的实现我们都是默认value是int，而且是正整数的int，然而生产中，不应该使用固定的value值。Java中应该使用泛型，GoLang中可以使用interface。

> 最大容量

上述我们的容器最大容量的单位是键值对的个数，这是不太合理的，因为实际中我们应该限制的是缓存占用大小，因此可以将最大限制改成byte为单位，而且需要对淘汰算法进行优化，这时候我们可能超出容量后，需要淘汰的不止是一个缓存，可以是多个，直到当前已用内存小于最大内存。

> 并发安全

上述我们写的只能在单线程下使用，没有考虑到并发问题，那么其实只需要对每次链表和队列的写查进行相应的加锁即可。Java可以使用synchronized关键字也可以使用ReentrantLock/ReentrantReadWriteLock来对其加锁。GoLang可以使用标准库的sync.Mutex来加锁。

> 其他

我们这次写的是每次都更新到首位的LRU，称为lru-1，也有lru-k的方式，这个需要根据情况进行优化。

### 优化后代码

针对上述的几个问题，我写了一个GoLang的优化后的LRU算法(仍然是lru-1)

```go
package lru

import (
   "container/list"
   "fmt"
)

type Cache struct {
   //缓存队列最大缓存大小
   maxBytes int64
   //缓存队列目前已经使用大小
   usedBytes int64
   //链表
   cacheList *list.List
   //map映射key->Element
   cacheMap map[string]*list.Element
   //删除记录时的回调函数
   OnEvicted func(key string, value Value)
}

//键值对类型
type entry struct {
   key   string
   value Value
}

//缓存的值的类型
type Value interface {
   //返回Value的内存大小
   Len() int
}

//实例化函数
func New(maxBytes int64, onEvicted func(string, Value)) *Cache {
   return &Cache{
      maxBytes:  maxBytes,
      usedBytes: 0,
      cacheList: list.New(),
      cacheMap:  make(map[string]*list.Element),
      OnEvicted: onEvicted,
   }
}

//查找
func (c *Cache) Get(key string) (Value, bool) {
   //先从map里查找是否有该值
   if element, ok := c.cacheMap[key]; ok {
      //若有则将该提至最近使用的
      c.cacheList.MoveToFront(element)
      //返回(将element的value强转成自定义的Value类型
      e := element.Value.(*entry)
      return e.value, true
   }
   return nil, false
}

//删除最久未被使用的节点
func (c *Cache) RemoveOldest() {
   //获取最后一个节点
   cur := c.cacheList.Back()
   if cur != nil {
      //从list中删除
      c.cacheList.Remove(cur)
      //从map中删除
      entry := cur.Value.(*entry)
      delete(c.cacheMap, entry.key)
      //更新当前已用内存
      c.usedBytes -= int64(len(entry.key)) + int64(entry.value.Len())
      //运行回调函数
      if c.OnEvicted != nil {
         c.OnEvicted(entry.key, entry.value)
      }
   }
}

//添加缓存
func (c *Cache) Put(key string, value Value) {
   //判断当前是否已经有该key
   if element, ok := c.cacheMap[key]; ok {
      //已经有该节点,则将该节点更新并移至队首(最近访问)
      c.cacheList.MoveToFront(element)
      entry := element.Value.(*entry)
      //更新当前已用内存(加上当前新增的value大小再减去原本的value大小)
      c.usedBytes += int64(value.Len()) - int64(entry.value.Len())
      //更新值
      entry.value = value
   } else {
      //若不存在,则添加至队首,并更新map
      element := c.cacheList.PushFront(&entry{key, value})
      //添加至map
      c.cacheMap[key] = element
      //更新已用内存
      c.usedBytes += int64(len(key)) + int64(value.Len())
   }
   //添加之后判断是否已经超过最大内存(当最大内存不为0时而且已用内存超过最大内存时删除最近未使用的节点)
   for c.maxBytes != 0 && c.maxBytes < c.usedBytes {
      c.RemoveOldest()
   }
}

//获取当前缓存的数据条数
func (c *Cache) Len() int {
   return c.cacheList.Len()
}

func (c *Cache) Print() {
   for cur := c.cacheList.Front(); cur != nil; cur = cur.Next() {
      kv := cur.Value.(*entry)
      fmt.Printf("%s->%s ", kv.key, kv.value)
   }
   fmt.Println()
}
```

```go
package gocache

import (
   "github.com/TheR1sing3un/gocache/gocache/lru"
   "sync"
)

type cache struct {
   //互斥锁
   lock sync.Mutex
   //lru缓存队列
   lru *lru.Cache
   //最大缓存大小
   cacheBytes int64
}

//缓存put方法
func (c *cache) put(key string, value ByteView) {
   c.lock.Lock()
   defer c.lock.Unlock()
   if c.lru == nil {
      //懒加载lru
      c.lru = lru.New(c.cacheBytes, nil)
   }
   c.lru.Put(key, value)
}

//缓存get方法
func (c *cache) get(key string) (value ByteView, ok bool) {
   c.lock.Lock()
   defer c.lock.Unlock()
   if c.lru == nil {
      //还未初始化(当前肯定没有数据)
      return
   }
   if value, ok := c.lru.Get(key); ok {
      return value.(ByteView), true
   }
   return
}
```

## 总结

至此，我们就完成了三种手写LRU算法，不需要过多练习，只需要记住核心思想就是每次get时候我就提前，每次put也要提前，并且需要判断是否满载，若满则删除最后的(最久未被访问的节点)。


