# MIT 6.830 Lab翻译


# MIT-6.830Lab翻译

## Lab1

在6.830的实验作业中，您将编写一个名为 `SimpleDB` 的基本数据库管理系统。对于这个lab，您将专注于实现访问存储在磁盘上的数据所需的核心模块; 在未来的lab中，您将添加对各种查询处理操作符以及事务、锁定和并发查询的支持。

`SimpleDB`使用Java语言写的。我们已经为您提供了一组大部分还未实现的类和接口。你将需要为这些类编写代码。我们将通过一组使用`JUnit`的系统测试来对代码进行打分。我们还提供了一些单元测试，我们不会将其用于评级，但是您可能发现再验证代码是否有效的时候十分有用。我们还鼓励你再我们的测试之外开发自己的测试的套件。

本文的其余部分描述了`SimpleDB`的基本架构，提供了一些关于如何开始编码的建议，并讨论了如何上交lab。

我们强烈建议你尽早开始写这个lab，它要求您编写大量的代码。

### 0. 环境设置

根据[指示](https://github.com/MIT-DB-Class/simple-db-hw-2021)从GitHub上下载lab1的代码。

这些指示是为Athena或者别的基于Unix的平台编写的(例如，Linux，MacOS等等)。因为代码是用Java写的，它在Windows下同样可以工作，即使文档中的指示可能不适用。

我们包含了使用Eclipse和IntelliJ来使用项目的[章节1.2](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab1.md#eclipse)。

### 1. 开始

`SimpleDB`使用[Ant build tool]()来编译代码和运行测试。Ant和[make]()相似，但是构建文件使用XML写的以及更适合Java代码。大多数现代Linux发行版包括Ant。在Athena下，它被包含在`sipd`储物柜中，你可以在Athena提示符下输入`add sipd`来得到它。注意，对于Athena的某些版本，您还必须运行`add -f java`来设置正确的Java环境。详细信息请查询[Athena使用Java的文档]()。

为了在开发过程中帮助您，除了用于评分的端到端测试外，我们还提供了一组单元测试。这些都不是全面的，你不应该仅靠它们来验证项目的正确性。(使用6.170的技巧！)

使用`test`来构建目标来运行单元测试：

```shell
$ cd [project-directory]
$ # run all unit tests
$ ant test
$ # run a specific unit test
$ ant runtest -Dtest=TupleTest
```

你应该看到相似的输出：

```shell
 build output...

test:
    [junit] Running simpledb.CatalogTest
    [junit] Testsuite: simpledb.CatalogTest
    [junit] Tests run: 2, Failures: 0, Errors: 2, Time elapsed: 0.037 sec
    [junit] Tests run: 2, Failures: 0, Errors: 2, Time elapsed: 0.037 sec

 ... stack traces and error reports ...
```

上面的输出表明在编译过程中出现了两个错误，这是因为我们给出的代码还不能工作。当您完成lab的各个部分时，您将努力的通过其他的单元测试。

如果您希望在编写代码的过程中编写新的单元测试，那么你应该将他加入到`test/simpledb`文件下。

对于更多实用Ant的细节，请看[说明手册](http://ant.apache.org/manual/)。[运行Ant](http://ant.apache.org/manual/running.html)章节提供更多的`ant`命令的细节。无论如何，下面快速参照表应该足以用于lab。

| Command                        | Description                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| ant                            | Build the default target (for simpledb, this is dist).       |
| ant -projecthelp               | List all the targets in `build.xml` with descriptions.       |
| ant dist                       | Compile the code in src and package it in `dist/simpledb.jar`. |
| ant test                       | Compile and run all the unit tests.                          |
| ant runtest -Dtest=testname    | Run the unit test named `testname`.                          |
| ant systemtest                 | Compile and run all the system tests.                        |
| ant runsystest -Dtest=testname | Compile and run the system test named `testname`.            |

如果您在 windows 系统下，不想从命令行运行 ant 测试，也可以从 eclipse 中运行它们。右键单击 build.xml，在目标标签中，你可以看到“ runtest”“ runsyest”等。例如，选择 runtest 将等效于命令行中的“ ant runtest”。可以在“ Main”选项卡“ Arguments”文本框中指定“-Dtest = testname”等参数。请注意，您还可以通过从 build.xml 中复制、修改目标和参数并将其重命名为 runtest _ build.xml 来创建一个 runtest 快捷方式。

#### 1.1运行端对端测试

我们也提供了一系列端对端的测试以及最终会被使用来打分。这些测试以`test/simpledb/systemtest`文件下的Junit测试组成。使用`systemtest`来构建目标来运行所有的系统测试。

```shell
$ ant systemtest

 ... build output ...

    [junit] Testcase: testSmall took 0.017 sec
    [junit]     Caused an ERROR
    [junit] expected to find the following tuples:
    [junit]     19128
    [junit] 
    [junit] java.lang.AssertionError: expected to find the following tuples:
    [junit]     19128
    [junit] 
    [junit]     at simpledb.systemtest.SystemTestUtil.matchTuples(SystemTestUtil.java:122)
    [junit]     at simpledb.systemtest.SystemTestUtil.matchTuples(SystemTestUtil.java:83)
    [junit]     at simpledb.systemtest.SystemTestUtil.matchTuples(SystemTestUtil.java:75)
    [junit]     at simpledb.systemtest.ScanTest.validateScan(ScanTest.java:30)
    [junit]     at simpledb.systemtest.ScanTest.testSmall(ScanTest.java:40)

 ... more error messages ...
```

这表示此测试失败，显示在何处检测到错误的堆栈跟踪。要进行调试，首先读取发生错误的源代码。当测试通过时，你会看到如下内容:

```shell
$ ant systemtest

 ... build output ...

    [junit] Testsuite: simpledb.systemtest.ScanTest
    [junit] Tests run: 3, Failures: 0, Errors: 0, Time elapsed: 7.278 sec
    [junit] Tests run: 3, Failures: 0, Errors: 0, Time elapsed: 7.278 sec
    [junit] 
    [junit] Testcase: testSmall took 0.937 sec
    [junit] Testcase: testLarge took 5.276 sec
    [junit] Testcase: testRandom took 1.049 sec

BUILD SUCCESSFUL
Total time: 52 seconds
```

##### 1.1.1创建虚拟表

您可能希望创建自己的测试和自己的数据表，以测试自己的 `SimpleDB` 实现。你可以创建任何`.txt`文件，并使用下列命令来将其转化为`SimpleDB`的`HeapFile`格式的`.dat`文件：

```shell
$ java -jar dist/simpledb.jar convert file.txt N
```

当文件的名字为`file.txt`以及`N`时文件的列数。注意`file.txt`必须是以下的格式：

```txt
int1,int2,...,intN
int1,int2,...,intN
int1,int2,...,intN
int1,int2,...,intN
```

其中iniN是个非负整数。

使用`print`命令来查看表的内容。

```shell
$ java -jar dist/simpledb.jar print file.dat N
```

这里的`file.dat`是使用`convert`命令创建出来的表的名字，以及`N`是文件中的列数。

#### 1.2使用IDE

略

#### 1.3实现提示

在开始写代码之前，我们强烈建议你先去阅读整个文档来了解对`SimpleDB`的顶层设计。

您将需要填写任何未实现的代码。我们认为您应该在哪里编写代码，这是显而易见的。您可能需要添加私有方法和/或帮助器类。你可以改变 api，但是要确保我们的[评分](https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab1.md#grading)测试仍然在运行，并且确保在你的记录中提到、解释和为你的决定辩护。

除了您需要为这个实验室填写的方法之外，类接口还包含许多您在后续实验之前不需要实现的方法。这些要么按类别列出:

```java
// Not necessary for lab1.
public class Insert implements DbIterator {
```

或者每个方法：

```java
public boolean deleteTuple(Tuple t)throws DbException{
        // some code goes here
        // not necessary for lab1
        return false;
        }
```

您提交的代码应该可以编译，而无需修改这些方法。

我们建议使用本文档的练习来指导您的实现，但是您可能会发现不同的顺序对您更有意义。

**下面是一个你可以继续使用 SimpleDB 实现的方法的大致轮廓:**

---

- 实现管理元组的类，即Tuple，TupleDesc。我们已经为您实现了Field，IntField，StringField和Type。因此您只需要支持integer以及(定长)string字段以及定长的tuples，这些都很简单
- 实现Catalog/目录(应该很简单)。
- 实现BufferPool(缓存池)的构造以及`getPage()`方法。
- 实现访问方法，HeapPage和HeapFile以及相关的ID类，这些文件中很大一部分已经为您编写了。
- 实现操作符SeqScan。
- 此时，您应该能够通过ScanTest系统测试，这是本lab的目标。

---

下面第二章节将更详细地向您介绍这些实现步骤以及与每个步骤对应的单元测试。

#### 1.4事务，锁，恢复

当你通过我们给你提供的接口上锁时，你将看到许多关于锁定、事务和恢复的引用。你在lab中不需要支持这些特性，但是你应该保持这些参数在代码的接口中，因为你在将来的lab中将会实现事务和锁定。测试代码给你提供了一个伪造的事务id，该id被传递到它运行的查询操作符中，您将这些事务id传递给其他的操作和缓冲池。

### 2. `SimpleDB`架构以及实现指引

`SimpleDB包括`：

- 表示字段、元组和元组模式的类。
- 对元组的应用谓词和条件的类。
- 一个或多个访问方法(例如，Heap files)在磁盘上的存储关系，并提供一个遍历这些关系的元组的方法。
- 处理元组的操作符的集合(例如，select，join，insert，delete等)。
- 一个缓冲池，在内存中缓存活跃的元组和页，并处理并发控制和事务(这两者在本lab中都不需要担心)。
- 存储有关可用表以及其架构的信息的目录。

`SimpleDB`不包括许多您可能认为是“数据库”一部分的内容，特别是，`SimpleDB`没有：

- (在本lab)一个SQL前端或解析器，允许您将查询直接输入到`SimpleDB`中。相反，查询是通过一组运算符连接到一个手工构建的查询计划中来构建的(参考[章节2.7]())。我们将为以后的lab提供一个简单的解析器。
- 视图。
- 除了整数和定长字符串以外的数据类型。
- (本lab中)查询优化器。
- (本lab中)索引。

在本节的其余部分中，我们将描述在这个lab中需要实现的`SimpleDB`的每一个主要组件。您应该使用本讨论中的练习来指导您的实现。本文档绝不是`SimpleDB`的完整规范，您需要决定如何设计和实现系统的各个部分。青丘狐注意，对于lab1，除了顺序扫描外，您不需要实现任何操作符(例如，select，join，project)。您在之后的Lab中会添加对其他操作的支持的。

#### 2.1 Database类

Database 类提供对作为数据库全局状态的静态对象集合的访问。特别是，这包括访问目录(数据库中所有表的列表)、缓冲池(当前驻留在内存中的数据库文件页的集合)和日志文件的方法。您不必担心本lab的日志文件。我们已经为您实现了 Database 类。您应该查看这个文件，因为您将需要访问这些对象。

#### 2.2 字段和元组

`SimpleDB`中的元组是十分基础的。它们由一组`Field`的对象组成，`Tuple`中每个字段一个对象。`Field`是不同数据类型(如，整数、字符串)实现的接口。元组对象是由底层访问方法(例如，heap files或b树)创建的，如下一节所述。元组还有一个类型(或者模式)，称为`_tuple`描述符`_`，由`TUpleDesc`对象表示。这个对象由一组`Type`对象组成，元组中每个字段一个，每个字段描述相应字段的类型。

#### Exercise 1

**在以下实现框架方法：**

---

- src/java/simpledb/storage/TupleDesc.java
- src/java/simpledb/storage/Tuple.java

---

此时，您的代码应该通过了`TupleTest`和`TupleDescTest`的单元测试。此时，`modifyRecordId()`应该会失败，因为您还没有实现它。

#### 2.3 目录

目录(`SimpleDB`中的`Catalog`类)包括由当前数据库中的表和表的模式的列表组成。你需要支持新增表的能力，同时从指定表获取信息的能力。与每个表关联的是一个`TupleDesc`对象，该对象允许运算符确定表中的字段的类型和数量。

全局的目录是为整个`SimpleDB`分配的一个`Catalog`的单例。可以通过`Database.getCatalog()`来检索全局目录，全局缓冲池也是如此。(使用`Database.getBufferPool()`)。

#### Exercise 2

**实现下述框架方法：**

---

- src/java/simpledb/common/Catalog.java

---

此时，你的代码应该通过了`CatalogTest`的单元测试。

#### 2.4 缓冲池

缓冲池(类`BufferPool`)负责缓存最近从磁盘读取的页到内存中。所有操作符在缓冲池中读写磁盘上不同的文件的页。它由固定数量的页组成，由`BufferPool`构造函数中的`numPages`参数定义。在以后的lab中，您将实现淘汰策略。对于该lab，您只需要实现SeqScan操作使用的构造函数和`BufferPool.getPage()`方法。缓冲池应该存储最多`numPages`个页面。对于这个lab，如果针对不同的页发出了超过`numPages`个请求，您可以抛出一个`DBExcaption`，而不是执行淘汰策略。后续的lab您会实现一个淘汰策略。

`Database`类提供一个静态方法，`Database.getBufferPool()`，它会返回对整个`SimpleDB`进程的缓冲池单例的引用。

#### Exercise 3

**实现下述框架方法：**

---

- src/java/simpledb/storage/BufferPool.java

---

我们没有给缓冲池提供单元测试。你实现的功能将在下面的HeapFile实现中被测试。你应该使用`DbFile.readPage`方法来访问DbFile的页面。

#### 2.5 堆文件访问方法

访问方法提供了一种从特定方式排列的磁盘读写数据的方法。常见的访问方法包括了堆文件(未排序的元组文件)和b树；对于这个分配，您将只实现堆文件的访问方法，我们已经为您编写了一些代码。

一个`HeapFile`对象被安排成一组页，每个页由固定数量的字节组成，用于存储元组(由常量`BufferPool.DEFAULT_PAGE_SIZE`定义)，包括了一个头结构。在`SimpleDB`中，数据库中的每个表都有一个`HeapFile`对象。`HeapFile`中的每一个页都被安排为一组插槽，每个插槽可以容纳一个元组(`SimpleDB`中给定表的元组都具有相同的大小)。除了这些插槽之外，每个页面还有一个头部，由位图组成。每个元组插槽由一个位。如果特定元组对应的位为1，代表元组有效；如果为0，代表元组无效(例如，已删除或者未初始化)。`HeapFile`对象的页类型为`HeapPage`。它实现了`Page`接口。页面存储在缓冲池中，但是由`HeapFile`类读写。

`SimpleDB`在磁盘上存储对文件的格式和他们在内存中存储对文件的格式大致相同。每个文件包含连续排列在磁盘上的页数据。每个页由一个或多个字节组成，这些字节表示头结构，后面跟着实际页面内容的_page_size_的字节。每个元组的内容需要_tuple size`_* 8 位，头部需要一位，因此一个页中可以容纳的元组数目是：

_tuples per page_ = floor((_page size_ * 8) / (_tuple size_ * 8 + 1))

其中，_tuple size_是页中tuple的大小(以字节为单位)。这里的想法是，每个元组都需要在页中增加一个存储位。我们计算一个页面中的位数(通过增加页面大小8)，然后用元组中的位数(包括这个额外的头部位)除以这个数量，会得到每页元组的数量。底层操作会舍入到最接近的元组的整数个数(我们不希望再页上存储部分元组)。

一旦我们直到 了每个页面的元组数量，存储头部所需的字节数就很简单了：

`headerBytes = ceiling(tupsPerPage/8)`

ceiling操作会舍入到最接近的整数字节数(我们从不存储低于整个字节的头信息)。每个字节的低位(最低有效位)表示文件中前面的插槽的状态。因此，第一个字节的最低位表示页中的第一个槽是否正在使用。第一个字节的第二个最低位表示页中的第二个槽是否正在使用，以此类推。另外，请注意，最后一个字节的高阶位可能不对应于文件中的实际插槽，因为插槽的数量的可能不是8的倍数。还要注意，所有的Java虚拟机都是[big-endian]()。

#### Exercise 4

**实现下述的框架代码：**

---

- src/java/simpledb/storage/HeapPageId.java
- src/java/simpledb/storage/RecordId.java
- src/java/simpledb/storage/HeapPage.java

---

即使你在lab1中不会直接使用它们，但是我们要求您在`HeapPage`中实现`getNumEmptySlots()`和`isSlotUsed()`方法。这需要在页头中推动位。你可能会发现查看`HeapPage`或者`src/simpledb/HeapFileEncoder.java`中提供的其他方法有助于理解页面的布局。

你还需要在页面的元组上实现一个迭代器，这可能涉及到一个辅助类或者数据结构。

此时，您的代码应该通过`HeapPageIdTest`、`RecordIDTest`和`HeapPageReadTest`中的单元测试。

在实现了`HeapPage`之后，您将在这个lab中为`HeapFile`编写方法，以计算文件中的页数并从文件中读取页。然后，您能够从存储在磁盘上的文件中获取元组。

#### Exercise 5

**实现下述的框架代码：**

---

- src/java/simpledb/storage/HeapFile.java

---

需要从磁盘中读取页，首先需要计算文件中的正确偏移量。提示：为了以任何偏移量读写页面，您需要对文件进行随机访问。从磁盘读取页面时，不应调用缓冲池的方法。

您还需要实现`HeapFile.iterator()`方法，它应该遍历堆文件中每个页的元组。迭代器必须使用`BufferPool.getPage()`方法来访问`HeapFile`中的页。这种方法将页面加载到缓冲池，并最终用于(在之后的lab中)实现基于锁定的并发控制和恢复。不要在`open()`调用中将整个表加载到内存中--这会导致非常大的表出现内存不足的错误。

此时，您的代码应该能够通过`HeapFileReadTest`单元测试。

#### 2.6 操作符

操作符负责查询计划的实际执行。它们实现关系代数的操作。在`SimpleDB`中，操作符是基于迭代器的；每个操作符都实现`DbIterator`接口。

操作符通过将低级别的操作符传递到较高级别的操作符的构造函数，将它们连接在一起，从而连接到一个计划中。计划叶上的特殊访问方法操作符负责从磁盘读取数据(因此它们下面没有任何操作符)。

在计划的顶部，与`SimpleDB`交互的程序简单地在根操作符上调用`getNext`；该操作符然后在其子操作符上调用`getNext`，以此类推，直到调用这些叶操作符。他们从磁盘中获取元组并将其传递到树中(作为`getNext`地返回参数)；元组以这种方式传播计划，直到它们从根输出或被计划中另一个操作符合并或者拒绝。

对于这个lab，您只需要实现一个`SimpleDB`操作符。

#### Exercise 6

**实现下述的框架代码：**

---

- src/java/simpledb/execution/SeqScan.java

---

该操作符顺序的扫描构造函数中的由`tableid`指定的表页中的所有元组。该操作符应该通过`DbFile.iterator()`方法访问元组。

此时，您应该能够完成了`ScanTest`系统测试了。

你将在随后的lab中完善其他操作符。

#### 2.7 一个简单的查询

本节的目的是演示如何将这些不同的组件连接在一起来处理一个简单的查询。

假设您有一个数据文件“ some _ data _ file.txt”，其内容如下:

```
1,1,1
2,2,2 
3,4,4
```

你可以转化把它转化成一个`SimpleDB`可以查询的二进制文件：

````shell
java -jar dist/simpledb.jar convert some_data_file.txt 3
````

这里的参数"3"意思是输入有三列。

下面的代码实现了对该文件的简单选择查询。这段代码相当于SQL语句中的`SELECT * FROM some_data_fiel`。

```java
package simpledb;
import java.io.*;

public class test {

    public static void main(String[] argv) {

        // construct a 3-column table schema
        Type types[] = new Type[]{ Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE };
        String names[] = new String[]{ "field0", "field1", "field2" };
        TupleDesc descriptor = new TupleDesc(types, names);

        // create the table, associate it with some_data_file.dat
        // and tell the catalog about the schema of this table.
        HeapFile table1 = new HeapFile(new File("some_data_file.dat"), descriptor);
        Database.getCatalog().addTable(table1, "test");

        // construct the query: we use a simple SeqScan, which spoonfeeds
        // tuples via its iterator.
        TransactionId tid = new TransactionId();
        SeqScan f = new SeqScan(tid, table1.getId());

        try {
            // and run it
            f.open();
            while (f.hasNext()) {
                Tuple tup = f.next();
                System.out.println(tup);
            }
            f.close();
            Database.getBufferPool().transactionComplete(tid);
        } catch (Exception e) {
            System.out.println ("Exception : " + e);
        }
    }

}
```

该表有三个整数字段。为了表示出来，我们创建了一个`TupleDesc`的对象然后传递给它一个`Type`数组对象，以及一个`String`类型的字段名数组。一旦我们创建了这个`TupleDesc`，我们就初始化一个`HeapFile`对象来表示存储在`some_data_file.dat`中的表。一旦我们完成了表的创建，我们将其加入到目录中。如果这是一个已经在运行的数据库服务器，我们将加载这个目录信息。我们需要显式地加载它，以使这段代码被包含进去。

初始化完数据库系统后，我们创建一个查询计划。我们的计划只包含从磁盘扫描元组的 `SeqScan` 操作符。一般来说，这些运算符通过引用适当的表(在 `SeqScan` 中)或子运算符(在 Filter 中)来实例化。然后测试程序在 `SeqScan` 操作符上反复调用 `hasNext` 和 `next`。由于元组是从 `SeqScan` 输出的，所以它们在命令行上被打印出来。

我们**强烈建议**你尝试这个有趣的端到端测试。它将帮助你获得为`SimpleDB`编写测试程序的经验。您应该在src/java/simpledb目录创建文件"test.java"并写上代码，以及添加一些"import"语句并且在顶级目录下放置一些`some_data_file.dat`文件。然后运行：

```shell
ant
java -classpath dist/simpledb.jar simpledb.test
```

注意`ant`编译`test.java`然后生成一个新的包含它的jar文件。

### 3. 后续工作

略

---

---

---


