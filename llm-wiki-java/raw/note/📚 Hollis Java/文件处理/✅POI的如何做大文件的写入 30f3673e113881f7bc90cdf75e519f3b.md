# ✅POI的如何做大文件的写入

所有者: junk01

# ✅POI的如何做大文件的写入

> 原文：[https://www.yuque.com/hollis666/asgng6/kalmkdx5fukxt13q](https://www.yuque.com/hollis666/asgng6/kalmkdx5fukxt13q)
> 

# 典型回答

上一篇中介绍了POI的内存溢出以及几种Workbook，那么，我们在做文件写入的时候，该如何选择呢？他们的在内存使用上有啥差异呢？

我们接下来分别使用XSSFWorkbook和SXSSFWorkbook来写入一个Excel文件，分别看一下堆内存的使用情况。

### 使用XSSF写入文件

运行main方法的过程中，通过arthas看一下堆内存的使用情况：

执行memory命令（这个执行的时间点很重要，我是在`String filename = "example.xlsx";`前输出了一行日志，然后sleep 50s，我在控制台看到这行日志之后开始查看堆内存情况）：

得到结果：

即占用堆内存1200+M。

### 使用SXSSFWorkbook写入文件

同样通过Arthas查看内存占用情况：

占用内存在148M左右。

### 对比结果

同样的一份文件写入，XSSFWorkbook需要1200+M，SXSSFWorkbook只需要148M。所以大文件的写入，使用SXSSFWorkbook是可以更加节省内存的。

如果不方便使用arthas，也可以直接在JVM启动参数中增加Xmx150m的参数，运行以上两段代码，使用XSSFWorkbook的会抛出OOM：

而使用SXSSFWorkbook时则不会。

所以，在使用POI时，如果要做大文件的写入，建议使用SXSSFWorkbook，会更加节省内存。

# 扩展知识

## 为啥SXSSFWorkbook占用内存更小?