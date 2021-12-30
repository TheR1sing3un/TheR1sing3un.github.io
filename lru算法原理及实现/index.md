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

## 总结

至此，我们就完成了两种方法手写LRU算法，不需要过多练习，只需要记住核心思想就是每次get时候我就提前，每次put也要提前，并且需要判断是否满载，若满则删除最后的(最久未被访问的节点)。


