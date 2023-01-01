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

## Lab 2:SImpleDB Operators

> SimpleDB 操作符

在这个lab任务中，您将为 SimpleDB 编写一组操作符来实现表修改(例如，插入和删除记录)、选择、连接和聚合。它们将建立在您在实验1中编写的基础之上，为您提供一个可以对多个表执行简单查询的数据库系统。

此外，我们忽略了lab1中的缓冲池管理问题: 我们没有处理在数据库生存期中引用超出内存容量的页面时出现的问题。在 Lab 2中，您将设计一个回收策略来清空缓冲池中的陈旧页面。

您不需要在这个lab中实现事务或锁定。

本文的其余部分给出了一些关于如何开始编码的建议，描述了一组帮助您完成实验的练习，并讨论了如何交代码。这个实验室要求您编写大量的代码，所以我们鼓励您尽早开始！

### 1. 开始

#### 1.1 开始Lab 2

略

#### 1.2 实现提示

开始之前，我们**强烈鼓励**你去阅读完整的文档来对SImpleDB的顶层设计有一个感知。

我们建议您通过使用本文档的练习来指导您的实现，但是您可能会发现不同的顺序对您更你有意义。和以前一样，我们将通过查看代码并验证您已经通过了ant的目标`test`和`systemtest`。注意，代码只需要通过我们在这个lab中指定的测试，而不是所有的单元和系统测试。请查阅章节3.4来了解关于打分的完整讨论以及您需要通过的测试列表。

下面是一个关于你可以继续您的SimpleDB实现的粗略大纲，关于这个大纲中步骤的更多细节以及练习，将在下面的第二节给出。

- 实现操作符`Filter`和`Join`，并验证它们的测试是否有效。这些操作符的Javadoc注释中包含了有关它们如何工作的详细信息。我们已经给出了`Project`和`OrderBy`的实现了，它们可以帮助您理解其他操作符是如何工作的。
- 实现`IntegerAggregator`和`StringAggregator`。在这里，您将编写在一个输入元组序列中对跨多个组的特定字段实际计算聚合的逻辑。使用整数除法计算平均值，因为 SimpleDB 只支持整数。`StringAggegator` 只需要支持 COUNT 聚合，因为其他操作对字符串没有意义。
- 实现`Aggregate`操作符。和其他操作符一样，总计操作实现了`OpIterator`接口因此它们可以被放入到SimpleDB的执行计划中。注意`Aggregate`操作符的输出是每次调用`next()`时整个组的聚合值，聚合构造函数接收聚合和分组等字段。
- 实现关于元组的插入，删除以及`BufferPool`中的页面回收算法。你在这里不需要担心事务。
- 实现`Insert`和`Delete`操作符。就像所有的操作符一样，`Insert`和`Delete`实现了`OpIterator`，接收一个元组流来插入或者删除以及输出带有整数字段的单个元组，该元组表示插入或删除的元组数量。

注意SimpleDB没有实现任何一种的一致性或者完整性检查，因此可以将重复的记录插入到文件中，而且没有办法强制执行主键或者外键约束。

此时你应该确保你可以通过ant的目标`systemtest`。

你也可以使用提供的SQL解析器来运行以来你的数据库的SQL查询。查看[章节2.7]()的简单教程。

最后，您可能会注意到这个实验室中的迭代器继承自 `Operator` 类，而不是实现 `OpIterator` 接口。因为 `next`/`hasNext` 的实现经常是重复的、烦人的和容易出错的，所以 `Operator` 通用地实现了这个逻辑，只需要您实现一个更简单的 `readNext`。请随意使用这种类型的实现，或者只是实现 `OpIterator` 接口(如果您愿意的话)。为了实现 `OpIterator` 接口，从迭代器类中删除 `extends Operator`，代之以 `implements OpIterator`.

### 2. SimpleDB架构以及实现指导

#### 2.1 Filter和Join操作符

回想一下SimpleDB的`OpIterator`类实现了关系代数的操作。现在，您将实现两个操作符，这两个操作符比简单的表扫描稍微有趣一点。

- _Filter_：这个操作符只返回满足`Predicate`的元组，`Predicate`被指定为其构造函数的一部分。因此，它会过滤掉任何和谓词不匹配的元组。
- _Join_：这个操作符根据`JoinPredicate`将其两个子元组连接起来，`JoinPredicate`作为其构造函数的一部分传入。我们只需要一个简单嵌套循环连接，但是您可以探索更有趣的连接实现。描述您在lab中的实现。

#### Exercise 1

实现如下的框架方法：

---

- src/java/simpledb/execution/Predicate.java
- src/java/simpledb/execution/JoinPredicate.java
- src/java/simpledb/execution/Filter.java
- src/java/simpledb/execution/Join.java

---

此时，您的代码应该可以通过`PredicateTest`，`JoinPredicateTest`，`FilterTest`以及`JoinTest`。更多的，你应该确保你可以通过`FilterTest`和`JoinTest`系统测试。

#### 2.2 Aggregates操作符

另一个 SimpleDB 操作符使用 `GROUP BY`子句实现基本 SQL 聚合。您应该实现五个 SQL 聚合(`COUNT`、 `SUM`、 `AVG`、 `MIN`、 `MAX`)并支持分组。您只需要在单个字段上支持聚合，并按单个字段进行分组。

为了计算聚合，我们使用了一个 `Aggregator` 接口，它将一个新的元组合并到一个聚合的现有计算中。聚合器在构造过程中被告知应该使用什么操作进行聚合。随后，客户端代码应该为子迭代器中的每个元组调用 `Aggregator.mergeTupleIntoGroup()`。合并所有元组后，客户端可以检索聚合结果的 `OpIterator`。结果中的每个元组都是一对表单(`groupValue`、 `aggregateValue`) ，除非按字段分组的值是 `Aggregator.NO_GROUPING`，在这种情况下，结果是表单的单个元组`(aggregateValue)`。

注意，这种实现需要不同组的空间时线性的。对于本实验，你不需要担心组数量超过可用内存的情况。

#### Exercise 2

实现下述的框架代码：

---

- src/java/simpledb/execution/IntegerAggregator.java
- src/java/simpledb/execution/StringAggregator.java
- src/java/simpledb/execution/Aggregate.java

---

此时，您的代码应该可以通过`IntegerAggregatorTest`，`StringAggregatorTest`和`AggregateTest`。更多的，你应该可以通过`Aggregate`系统测试。

#### 2.3 堆文件可变性

现在，我们将开始实现支持修改表的方法。我们从单个页面和文件的级别开始。有两组主要的操作: 添加元组和删除元组。

删除元组: 要删除元组，您需要实现 `deletettuple`。元组包含 `recordid`，它允许您查找它们所驻留的页面，因此这应该很简单，只需查找元组所属的页面并适当地修改页面的标题即可。

添加元组: `HeapFile.java` 中的 `insertTuple` 方法负责向堆文件添加元组。要向 `HeapFile` 添加新的 tuple，您必须找到一个带有空槽的页面。如果 `HeapFile` 中不存在这样的页，则需要创建一个新页并将其附加到磁盘上的物理文件。您需要确保元组中的 `RecordID` 被正确更新。

#### Exercise 3

实现下述的框架代码：

---

- src/java/simpledb/storage/HeapPage.java
- src/java/simpledb/storage/HeapFile.java
  (注意你没有必要实现`writePage`方法)

---

为了实现HeapPage，你需要修改你的头部位图为了方法比如说`insertTuple()`和`deleteTuple()`。你可能发现lab 1中要求你实现的`getNumEmptySlots()`和`isSlotUsed()`方法可能会有用。注意，有一个`marySlotUsed()`方法作为一个抽象用来修改页头中的元组的填充或者清除状态。

注意，一个重要的点是`HeapFile.insertTuple()`方法和`HeapFile.deleteTuple()`方法是使用`BufferPool.getPage()`来访问页的；另外，你下一个lab中的事务实现不能正常工作。

实现`src/simpledb/BufferPool.java`中的框架方法：

---

- insertTuple()
- deleteTuple()

---

这些方法应该调用HeapFile中属于正在修改的表的适当方法(这种额外的间接级别用于支持其他类型的文件(比如索引))。

此时，你的代码应该可以通过`HeapPageWriteTest`和`HeapFileWriteTest`的单元测试，以及`BufferPoolWriteTest`。

#### 2.4 插入和删除

现在你已经写了所有用于添加和删除元组的HeapFile机制，接下来将实现`Insert`和`Delete`操作符。对于实现了`insert`和`delete`查询的计划，最上面的操作符是一个特殊的`Insert`和`Delete`操作符，用于修改磁盘上的页。这些操作符返回受影响的元组的数量。这是返回带有一个整数字段的单个元组实现了，该字段包含计数。

- _Insert_：这个操作符将其从子操作符中读取的元组添加到其构造函数中指定的`tableid`。它应该使用`BufferPool.insertTuple()`方法来完成。
- _Delete_：这个操作符将其从构造函数中指定的表中删除从其自操作符中读取的元组。它应该使用`BufferPool.deleteTuple()`方法来完成。

#### Exercise 4

实现下述的框架代码：

---

- src/java/simpledb/execution/Insert.java
- src/java/simpledb/execution/Delete.java

---

此时，你的代码应该可以通过`InsertTest`。我们没有提供`Delete`的单元测试。更多的额，你应该可以通过`InsertTest`和`DeleteTest`的系统测试。

#### 2.5 页面回收

在lab1中，在实验1中，我们没有正确地遵守由构造函数参数 `numPages` 定义的缓冲池中的最大页数限制。现在，您将选择一个页面收回策略，并检测以前读取或创建页面以实现策略的任何代码。

当缓冲池中有超过 `numPages` 页面时，应该在加载下一个页面之前将其中一个页面从缓冲池中删除。驱逐政策的选择取决于你; 没有必要做一些复杂的事情。描述一下你在该lab中的策略。

请注意，`BufferPool` 要求您实现一个 `flushAllPages()`方法。在实际的缓冲池实现中，这并不是您所需要的。但是，我们需要这个方法来进行测试。永远不要在任何实际代码中调用此方法。

由于我们实现 `ScanTest.cacheTest` 的方式，您需要确保您的 `flushPage` 和 `flushhallpages` 方法不会从缓冲池中排除页面，以便正确地通过这个测试。

`flushAllPages` 应该在 `BufferPool` 中的所有页面上调用 `flushPage`，而 `flushPage` 应该将任何脏页面写入磁盘并将其标记为非脏页面，同时将其保留在 `BufferPool` 中。

唯一应该从缓冲池中删除页面的方法是 `evictPage`，它应该在它清除的任何脏页面上调用 `flushPage`。

#### Exercise 5

填充方法`flushPage()`以及在以下方面实施页面驱逐的其他帮助方法：

---

- src/java/simpledb/storage/BufferPool.java

---

如果你在之前没有实现`HeapFile.java`中的`writePage()`，你将需要在此实现。最终，你应该也实现`discardPage()`来不刷新该页到磁盘就丢弃。我们将不会在本lab中测试`discardPage()`，但是这个在将来的lab中会很有必要。

此时，你的代码应该通过`EvictionTest`系统测试。

因为我们不会检查任何特定的收回策略，所以这个测试通过创建一个16页的 `BufferPool` 来工作(注意: `DEFAULT _ PAGES` 是50，而我们初始化 `BufferPool` 时会使用更少的页数)，扫描一个超过16页的文件，看看 JVM 的内存使用是否增加了超过5mb。如果您不能正确执行驱逐政策，您将无法驱逐足够的页面，并且将超过大小限制，因此无法通过测试。

#### 2.6 预查询

接下来的代码实现了一个简单两个表之间的`join`查询，每一个包括三个整数列(`some_data_file1.dat`和`some_data_file2.dat`是这个文件中页面的二进制表示)。代码相当于如下SQL语句：

```sql
SELECT *
FROM some_data_file1,
     some_data_file2
WHERE some_data_file1.field1 = some_data_file2.field1
  AND some_data_file1.id > 1
```

对于更广泛的查询操作示例，您可能会发现浏览关于连接、筛选器和聚合的单元测试很有帮助。

```java
package simpledb;

import java.io.*;

public class jointest {

    public static void main(String[] argv) {
        // construct a 3-column table schema
        Type types[] = new Type[]{Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE};
        String names[] = new String[]{"field0", "field1", "field2"};

        TupleDesc td = new TupleDesc(types, names);

        // create the tables, associate them with the data files
        // and tell the catalog about the schema  the tables.
        HeapFile table1 = new HeapFile(new File("some_data_file1.dat"), td);
        Database.getCatalog().addTable(table1, "t1");

        HeapFile table2 = new HeapFile(new File("some_data_file2.dat"), td);
        Database.getCatalog().addTable(table2, "t2");

        // construct the query: we use two SeqScans, which spoonfeed
        // tuples via iterators into join
        TransactionId tid = new TransactionId();

        SeqScan ss1 = new SeqScan(tid, table1.getId(), "t1");
        SeqScan ss2 = new SeqScan(tid, table2.getId(), "t2");

        // create a filter for the where condition
        Filter sf1 = new Filter(
                new Predicate(0,
                        Predicate.Op.GREATER_THAN, new IntField(1)), ss1);

        JoinPredicate p = new JoinPredicate(1, Predicate.Op.EQUALS, 1);
        Join j = new Join(p, sf1, ss2);

        // and run it
        try {
            j.open();
            while (j.hasNext()) {
                Tuple tup = j.next();
                System.out.println(tup);
            }
            j.close();
            Database.getBufferPool().transactionComplete(tid);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

两个表都有三个整数字段。为了表示这一点，我们创建一个 `TupleDesc` 对象，并向其传递一个 `Type` 对象数组，该数组指示字段类型和`String`对象，前者指示字段名称。一旦创建了这个 `TupleDesc`，我们就初始化两个表示表的 `HeapFile` 对象。创建表之后，将它们添加到 `Catalog` 中。(如果这是一个已经在运行的数据库服务器，我们将加载这个目录信息; 我们只需要为了这个测试的目的而加载这个目录信息)。

初始化完数据库系统后，我们创建一个查询计划。我们的计划由两个 `SeqScan` 操作符组成，它们扫描磁盘上每个文件的元组，连接到第一个 `HeapFile` 上的 `Filter` 操作符，连接到一个 `Join` 操作符，该操作符根据 `JoinPredicate` 将表中的元组连接起来。通常，这些运算符通过引用适当的表(对于 `SeqScan`)或子运算符(对于` Join`)来实例化。然后，测试程序在 `Join` 操作符的下一个位置上重复调用，然后 `Join` 操作符从其子元组中提取元组。由于元组是 `Join` 的输出，所以它们在命令行上被打印出来。

#### 2.7 查询解析器

我们为您提供了一个SimpleDB查询解析器，一旦您完成了本实验室的练习，您就可以使用该解析器对数据库编写和运行 SQL 查询。

第一步是创建一些数据表和目录。假设您有一个文件 `data.txt`，其内容如下:

```
1,10
2,20
3,30
4,40
5,50
5,50
```

您可以使用 `convert` 命令将其转换为SimpleDB的表(请确保首先输入 `ant`!) ：

```shell
java -jar dist/simpledb.jar convert data.txt 2 "int,int"
```

这会创建一个`data.dat`文件。除了表中的原始数据，这两个附加参数还指定了每个记录有两个字段，他们的类型是`int`和`int`。

接下来，创建一个目录文件，`catalog.txt`，携带如下内容：

```shell
data(f1 int, f2 int)
```

这告诉SimpleDB有一个表，`data`(存储在`data.dat`)的两个整数字段名为`f1`和`f2`。

最终，调用解析器。你必须从命令行运行java程序(ant对于交互式的目标不能正常工作)。从`simpledb/`文件下，输入：

```shell
java -jar dist/simpledb.jar parser catalog.txt
```

你应该看到如此输出：

```shell
Added table : data with schema INT(f1), INT(f2), 
SimpleDB> 
```

最终，你可以运行一个查询：

```shell
SimpleDB> select d.f1, d.f2 from data d;
Started a new transaction tid = 1221852405823
 ADDING TABLE d(data) TO tableMap
     TABLE HAS  tupleDesc INT(d.f1), INT(d.f2), 
1       10
2       20
3       30
4       40
5       50
5       50

 6 rows.
----------------
0.16 seconds

SimpleDB> 
```

解析器的功能相对完整(包括对 `SELECT`、 `INSERT`、 `DELETE` 和事务的支持) ，但是确实存在一些问题，并且不一定报告完全信息化的错误消息。这里有一些需要记住的限制:

- 即使字段名是唯一的，也必须在每个字段名前加上它的表名(如上面的例子所示，可以用表名的别名，但不能使用AS关键字)。
- `WHERE`子句支持嵌套查询，但`FROM`子句不支持嵌套查询。
- 不支持算数表达式(比如，你不可以拿到两个字段的合)。
- 最多允许一个`GROUP BY`和一个聚合列。
- 不支持面向集合的操作符比如说`IN`，`UNION`和`EXCEPT`。
- 只允许`WHERE`子句中的`ANT`表达式。
- 不支持`UPDATE`。
- 字符串操作符`LIKE`是支持的，但是必须写全(也就是说，不允许使用[~]简写)。

### 3. 后续工作

略

---

---

---

## Lab3: Query Optimization

> 查询优化

在这lab中，您将在 SimpleDB 之上实现一个查询优化器。主要任务包括实现选择性估计框架和基于成本的优化器。您可以自由决定具体实现什么，但是我们建议使用类似于课堂上讨论的 Selinger 基于成本的优化器(第9课)。

本文的其余部分描述了添加优化器支持所涉及的内容，并提供了如何添加优化器的基本概要。

和以前的lab一样，我们建议你尽早开始。

### 1. 开始

你应该在你完成了Lab 2的提交之后开始。

我们提供给你一个额外的测试用例以及本lab的源代码文件，这些文件不在您收到的最开始的发行版中。我们再次鼓励您在我们提供的测试套件之外开发自己的测试套件。

我们需要

You will need to add these new files to your release. The easiest way to do this is to change to your project directory (probably called simple-db-hw) and pull from the master GitHub repository:

```shell
$ cd simple-db-hw
$ git pull upstream master
```

#### 1.1 实现提示

我们建议使用本文档的练习来指导您的实现，但是您可能会发现不同的顺序对您更有意义。和以前一样，我们将通过查看您的代码并验证您已经通过了 ant 目标测试和系统测试的测试来为您的任务打分。关于评分和你需要通过的测试的完整讨论，请参阅第3.4节。

以下是本lab粗略的实现步骤。在后面的章节二中给出更多的具体细节。

- 实现`TableStats`类中的方法来允许它使用直方图(`IntHistogram`类提供的骨架)或者您设计的其他形式的统计数据来估计过滤器的选择性和扫描成本。
- 实现`JoinOptimizer`类中的方法来允许它估计`join`连接的成本和选择性。
- 编写`JoinOptimizer`中的`orderJoins`方法。这个方法在给定前两个的计算统计数据的情况下必须产生一个最优排序(可能使用Selinger算法)。

### 2. 优化器大纲

回想一下，基于成本的优化器的主要思想是:

- 使用表的统计信息来估计不同查询计划的“成本”。通常，计划的成本与中间联接和选择的基数(由中间联接产生的元组数量)以及筛选器和联接谓词的选择性有关。
- 使用这些统计数据以最佳方式排序连接和选择，并从多个备选方案中选择连接算法的最佳实现。

在本lab中，你将实现代码来完成上述的功能。

优化器将在`simpledb/Parser.java`中调用。你可能希望在开始这个lab之前回顾一下[lab 2 解析练习]()。简要的来讲，如果你有一个目录文件`catalog.txt`来表述你的表，那么你可以输入下述命令来解析：

```shell
java -jar dist/simpledb.jar parser catalog.txt
```

当解析器被调用时，它将计算所有表的统计信息(使用您提供的统计信息代码)。当请求查询时，解析器会将查询转化为逻辑计划表示，然后调用查询优化器来生成最优查询。

#### 2.1 总体优化器结构

在你开始这些实现之前，你需要了解SimpleDB的优化器的整体架构。图1表示SimpleDB的解析器和优化器的整体控制流程。

![_图1:说明解析器中使用的类、方法和对象的图_](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/controlflow.png)

底部的键解释了这些符号; 您将使用双边框实现这些组件。类和方法将在下面的文本中更详细地解释(您可能希望回到这个图) ，但是基本操作如下:

1. `Parser.java`初始化时构造了一组表统计信息(存储在`statsMap`容器中)，然后等待一个查询，并对该查询调用`parseQuery`。
2. `parseQuery`首先构造一个`LogicalPlan`代表解析后的查询。`parseQuery`然后调用方法构造出来的`LogicalPlan`实例的`physicalPlan`方法。`physicalPlan`方法返回一个可以被用在实际的查询中的`DBIterator`对象。

在即将到来的练习中，你将实现帮助`physicalPlan`设计一个最佳计划的方法。

#### 2.2 统计估计

准确估计计划成本是相当棘手的。在这个实验中，我们将只关注连接序列和基表访问的成本。我们不会担心访问方法的选择(因为我们只有一个访问方法，表扫描)或其他操作符(如聚合)的成本。

你只需要考虑这个lab的左深计划。有关您可能实现的其他“额外”优化器特性的描述，请参见第2.3节，其中包括处理复杂计划的方法。

##### 2.2.1 整体计划成本

我们将会编写从`p = t1 join t2 join ... tn`的join计划，它表示一个左深连接，其中t1是最左边的连接(树中最深的连接)。给定一个类似`p`的计划，它的成本可以如下表达：

```shell
scancost(t1) + scancost(t2) + joincost(t1 join t2) +
scancost(t3) + joincost((t1 join t2) join t3) +
... 
```

这里，`scancist(t1)`是扫描表t1的I/O成本，`joincost(t1, t2)`是连接j1到j2的CPU成本。为了让I/O成本和CPU成本可以比较，通常使用一个常量比例因子，例如：

```shell
cost(predicate application) = 1
cost(pageScan) = SCALING_FACTOR x cost(predicate application)
```

对于本lab你可以忽略缓存的影响(例如，假设每次访问一个表都要付出全表扫描的成本) -- 同样，这也是你可以在2.3章节中作为一个可选的附加扩展添加到你的lab中的东西。因此，`scancost(t1)`就是`t1 x SCALING_FACTOR`中的页数。

##### 2.2.2 连接成本

当使用嵌套循环连接时，回想一下两个表t1和t1(其中t1在外部)之间的连接成本是简单的：

```shell
joincost(t1 join t2) = scancost(t1) + ntups(t1) x scancost(t2) //IO cost
                       + ntups(t1) x ntups(t2)  //CPU cost
```

这里，`ntups(t1)`是表t1中的元组数量。

##### 2.2.3 过滤器选择性

通过扫描基表可以直接计算中`ntups`。对于一个包含一个或多个选择谓词的表进行`ntups`估计可能会比棘手 -- 这就是过滤器的选择性估计问题。这里有一个你可能会用到的方法，基于对表格中的值进行直方图计算：

- 计算每个表中的每个属性的最小值和最大值(扫描一次)。
- 为表中的每个属性建立一个直方图。一个方法是使用固定数量的桶 _NumB_，每个桶表示柱状图属性域的固定范围内的记录数量。例如，如果一个字段_f_的范围从1到100，并且有10个桶，那么桶1可能包含1到10之间记录数的计数，桶2包含11到20之间的记录数的计数，以此类推。
- 再次扫描表，选择出所有的元组的字段，并使用它们填充每个直方图中桶的计数。
- 要顾及等式表达式_f = const_的选择性，请计算包含值_const_的桶。假设桶的宽度(值的范围)为_w_，桶的高度(元组数量)为_h_，表中元组的数量为_ntups_。然后，假设值在桶中均匀分布，表达式的选择性大致为_(h / w) / ntups_，因为_(h / w)_表示值为_const_的桶中的预期元组数。
- 为了估计表达式_f > const_的选择性，计算_const_所在桶_b_，其宽为_w_b_，高度为_h_b_。然后，_b_包含整个元组的一个分数_b_f = h_b / ntups_。假设元组均匀分在_b_中，那么_b_中_> const_的部分是_(b_right - const) / w_b_，这里的_b_right_是桶_b_的右端点。因此，桶_b_为谓词的选择性贡献了_b_f x b_part_。此外，桶_b+1..NumB-1_贡献了了它们的全部的选择性(可以使用类似于上面_b_f_的公式来计算)。将所有的桶的选择性贡献相加得到表达式的总体选择性。图2说明了这个过程。
- 设计到_less than_的表示式的选择性可以执行类似于_greater than_的类似表达式，桶从0开始。

![_Figure2: 图解说明了会在lab5中实现的直方图_](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220528170350623.png)

---

##### Exercise 1：IntHistogram.java

你需要实现某些方法来记录表中的统计信息以及进行选择性评估。我们已经提供了框架类，`IntHistogram`来实现这一点。我们的目的是使用上面描述的基于桶的方法计算直方图，但是您可以随意使用其他的方法，只要它提供合理的选择性估计。

我们提供给你一个类`StringHistogram`来使用`IntHistogram`来计算字符串谓词的选择性。你可能需要修改`StringHistogram`如果你想要实现一个更好的估计器，但是为了完成这个lab您不需要这么做。

在完成了这个练习之后，你应该可以通过`IntHistogramTest`单元测试(如果你选择不实现基于直方图的选择性估计那么你不被要求通过这个测试)。

---

##### Exercise 2：TableStats.java

`TableStats`类包含一些方法，这些方法计算表中的元组和页数，并估计谓词对该表字段的选择性。我们已经创建的解析器会给每个表创建一个`TableStats`实例，然后传递这些结构到你的查询优化器(你在之后的练习中会需要)。

你应该填充下下述的方法以及`TableStats`中的类：

- 实现`TableStats`构造函数：一旦你已经实现一个方法来用于跟踪类似于直方图之类的统计信息，你应该实现`TableStats`构造函数，添加到代码去扫描表(可能多次)来构建你需要的统计信息。
- 实现`extimateSelectivity(int field, Predicate.Op op, Field constant)`：使用你的统计信息(例如一个取决于变量的类型的`IntHistogram`或`StringHistogram`)，估计表中谓词`field op constant`的选择性。
- 实现`estimateScanCost()`：这个方法估计线性扫描文件的成本，给定扫描一页的成本是`costPerPageIO`。你可以假设没有找寻以及缓冲池中没有页。这个方法可以使用在构造函数中计算成本或者大小。
- 实现`estimateTableCardinality(double selectivityFactor)`：如果应用具有选择性的谓词，这个方法返回关系中的元组数。这个方法可能使用在您在构造函数中计计算成本或大小。

你可能希望修改`TableStats.java`中的构造函数，例如，为了选择性估计的目的，计算上面描述的字段的直方图。

在完成了这些任务之后你应该可以通过`TableStatsTest`中的单元测试

---

##### 2.2.4 连接基数

最后，注意上面的连接计划 `p` 的成本包括表单联合成本`((t1 join t2) join t3)`的表达式。为了计算这个表达式，您需要一些方法来估计 `t1 join t2`的大小。这个连接基数估计问题比过滤器选择性估计问题更难。在这个实验室中，您不需要为此做任何花哨的事情，尽管第2.4节中的一个可选练习包括一个基于直方图的连接选择性评估方法。

当你实现你的简单的实现的时候，你需要注意下述事项：

- 对于相等链接，当其中一个属性是主键的时候，连接生成的元组数量不能大于非主键属性的基数。
- 对于没有主键的等式连接，很难说输出的大小是多少--它可以是表基数乘积的大小(如果两个表中所有元组都有相同的值)--或者可以是0。构建一个简单的启发式方法(比如，两个表中较大的表的大小)是可行的。
- 对于范围扫描，很难说任何准确的大小。输出的大小应该与输入的大小成正比。假设范围扫描(例如说，30%)产生的大小是可以固定的。一般来说，范围连接的成本应该大于大小相同的两个表的非主键连接成本。

---

##### Exercise 3：Join Cost Estimation

> Join成本估计

类`JoinOptimizer.java`包括用于排序和计算连接成本的所有方法。在本练习中，您将编写用于评估连接的选择性和成本的方法，具体如下：

- 实现`estimateJoinCost(LogicalJoinNode j, int card1, int card2, double cost1, double cost2)`：这个方法估计`join j`的成本，假设左输入的成本是card1，右输入的成本是card2，扫描左输入的成本是cost1，扫描右输入的成本是cost2。你可以假设连接是NL join，并应用之前提到的公式。
- 实现`estimateJoinCardinality(LogicalJoinNode j, int card1, int card2, boolean t1pkey, boolean t2pkey)`：这个方法通过`join j`估计出元组输出的数量，假设左边的输入card1的大小，右边输入的是card2的大小，标志位t1pkey和t2pkey分别表示左边和右边的字段是否唯一(主键)。

在实现了这些方法之后，你应该可以通过单元测试`JoinOptimizerTest.java`中的`estimateJoinCostTest`和`estimateJoinCardinality`测试。

---

#### 2.3 连接排序

现在你已经实现了估计成本的方法，你将实现Selinger优化器。对于这些方法，joins被表示为连接节点列表(例如，两个表上的谓词)，而不是类中描述的要连接的关系列表。

将课程中给出的算法转换为上面提到的连接节点列表形式，伪代码的大纲如下:

```c
1. j = set of join nodes
2. for (i in 1...|j|):
3.     for s in {all length i subsets of j}
4.       bestPlan = {}
5.       for s' in {all length d-1 subsets of s}
6.            subplan = optjoin(s')
7.            plan = best way to join (s-s') to subplan
8.            if (cost(plan) < cost(bestPlan))
9.               bestPlan = plan
10.      optjoin(s) = bestPlan
11. return optjoin(j)
```

为了帮助您实现这个算法，我们提供了几个类和方法来帮助您。首先，`JoinOptimizer.java` 中的`enumerateSubsets(List v，int size)`方法将返回一组大小为 `v` 的所有子集。这种方法对于大型集合来说效率非常低; 您可以通过实现更高效的枚举器来获得额外的分数(提示: 考虑使用就地生成算法和惰性迭代器(或流)接口来避免实现整个功率集)。

然后，我们提供一个方法：

```java
    private CostCard computeCostAndCardOfSubplan(Map<String, TableStats> stats, 
                                                Map<String, Double> filterSelectivities, 
                                                LogicalJoinNode joinToRemove,  
                                                Set<LogicalJoinNode> joinSet,
                                                double bestCostSoFar,
                                                PlanCache pc) 
```

给定一个联接子集`(joinSet)`和一个要从该集中移除的联接`(joinToRemove)` ，此方法计算将 `joinToRemove` 联接到 `joinSet-{ joinToRemove }`的最佳方法。它在 `CostCard` 对象中返回这个最佳方法，其中包括成本、基数和最佳连接顺序(作为列表)。如果找不到计划(例如，因为不存在可能的左深连接) ，或者如果所有计划的成本大于` bestCostSoFar` 参数，则 `computeCostAndCardOfSubplan` 可能返回 null。该方法使用一个名为 `pc` (在上面的 伪代码中为 `optjoin`)的前联接缓存来快速查找联接 `joinSet-{ joToRemove }`的最快方法。其他参数(`stats` 和 `filterSelectivities`)被传递到 `orderJoins` 方法中，您必须将其作为练习4的一部分来实现，下面将对其进行解释。这个方法实际上执行前面描述的假代码的第6-8行。

在之后，我们提供方法：

```java
    private void printJoins(List<LogicalJoinNode> js, 
                           PlanCache pc,
                           Map<String, TableStats> stats,
                           Map<String, Double> selectivities)
```

此方法可用于显示连接计划的图形化表示(例如，当通过优化器的"-explain"选项设置标志时)。

最后，我们已经提供了一个类 `PlanCache`，它可以用来缓存到目前为止在 Selinger 实现中考虑的连接子集的最佳方式(使用 `computeCostAndCardOfSubplan` 需要该类的一个实例)。

---

##### Exercise 4：Join Ordering

> 连接排序

在`JoinOptimizer.java`中，实现下述方法：

```java
  List<LogicalJoinNode> orderJoins(Map<String, TableStats> stats, 
                   Map<String, Double> filterSelectivities,  
                   boolean explain)
```

此方法应对`joins`类成员进行操作，并返回一个新列表，该列表指定了联接的执行顺序。此列表的项0指示左深计划中最左、最底的联接。返回列表中的相邻联接应该至少共享一个字段，以确保计划是左深的。这里的 `stats` 是一个对象，可以通过它找到出现在查询的 `FROM` 列表中的给定表名的 `TableStats`。`filterSelectivities` 允许您找到任何谓词对表的选择性; 它保证在 `FROM` 列表中每个表名有一个条目。最后，`explain`指定为了提供信息，应该输出连接顺序的表示形式。

您可能希望使用上面描述的帮助器方法和类来帮助您的实现。粗略地说，您的实现应该遵循上面的伪代码，循环遍历子集大小、子集和子集的子计划，调用 `computeCostAndCardOfSubplan` 并构建一个 `PlanCache` 对象，该对象存储执行每个子集连接的最小成本方式。

实现此方法后，您应该能够通过 `JoinOptimizerTest` 中的所有单元测试。您还应该通过系统测试 `QueryTest`。

#### 2.4 格外的分数

在本节中，我们将描述您可以实现以获得额外学分的几个可选练习。与前面的练习相比，这些练习的定义不那么明确，但是它们可以让您有机会展示自己对查询优化的掌握！请在报告中清楚标明你选择完成的项目，并简要说明你的实施情况和结果(基准数字、经验报告等)

---

**加分练习**。这些格外加成中的每一项都可以获得高达5%的额外奖励。

- _添加代码以执行更高级的连接基数估计_。语气使用简单的启发式方法来估计连接基数，不如设计一个更复杂的算法。

  - 一种方法是在每一对表_t1_和_t2_中的每一对属性_a_和_b_之间使用联合直方图。这个想法是创建_a_的桶列，并为_a_的每个桶_A_创建_b_值的柱状图，这些值和_A_桶中的值_a_同时出现。
  - 估计联接基数的另一种方法是假设较小表中的每个值在较大表中都有匹配值。那么连接选择性的公式是: _1/(Max (num-distinct (t1，column1) ，num-distinct (t2，column2))))_。这里，column1和 column2是联接属性。连接的基数是 t1和 t2的基数乘以选择性的产物。

- _改进子集迭代器_。我们的`enumerateSubsets`实现十分低效，因为它在每个调用上创建大量的Java对象。

  在这个格外的练习中，你将提升`enumerateSubsets`的性能，以至于你的系统可以对包含了20个或者更多连接的计划执行查询优化(当前这种计划需要花费数分钟或者数小时来进行计算)。

- _一个支持缓存的成本模型_。估计扫描和连接成本的方法不考虑缓冲池中的缓存。您应该扩展成本模型以考虑缓存效果。这是很棘手的，因为由于迭代器模型，多个连接同时运行，所以很难预测每个连接使用我们在以前实验室中实现的简单缓冲池可以访问多少内存。

- _改进join算法以及算法的选择_。我们当前的成本估算和连接算子选择算法(请参阅 `JoinOptimizer.java` 中的 `instanatejoin ()`)只考虑嵌套循环连接。扩展这些方法以使用一个或多个附加的连接算法(例如，使用 `HashMap` 进行某种形式的内存散列)

- _密集计划_。改进所提供的 `orderjoin ()`和其他助手方法，以生成密集的连接。我们的查询计划生成和可视化算法完全能够处理杂乱的计划; 例如，如果 `orderjoin ()`返回列表`(t1 join t2; t3 join t4; t2 join t3)` ，这将对应于一个杂乱的计划，顶部有`(t2 join t3)`节点。

---

你现在可以完成这个lab了。

### 3. 后续工作

略

---

---

---


