# ✅给你一个文本文件，每一行包含一个 QQ号码，请用linux命令进行去重？

所有者: junk01

# ✅给你一个文本文件，每一行包含一个 QQ号码，请用linux命令进行去重？

> 原文：[https://www.yuque.com/hollis666/asgng6/kecaf3dy10hhkcog](https://www.yuque.com/hollis666/asgng6/kecaf3dy10hhkcog)
> 

# 典型回答

想要在linux文件中移除重复的行，可以用`sort -u`命令：

- `sort`：将文件的内容按字典顺序排序。
- `-u`：表示去重（unique），会在排序后移除重复的行。
- `filename.txt`：是包含 QQ 号码的文件名。
- `-o filename.txt`：将去重后的内容直接写回到原文件。