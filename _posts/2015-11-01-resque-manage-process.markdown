---
layout: post
title:  "作业队列Resque如何管理进程"
date:   2015-11-01 23:29:25
categories: technology
author: tank
---

## 架构

[手打此文部分参考unix进程]Resque是一个基于Redis的库，用于创建后台作业，将这些作业放入多个队列并随后处理。

Resque worker负责随后处理这一环节。它的工作是启动并载入应用程序环境，然后连接Redis并设法保留任何尚未处理的后台作业。如果能够保留此类作业，它就将该作业从队列中清除，然后返回第一步。

对于大型应用来说，一个Resque worker是不够的。因此很常见的做法是利用多个Resque worker并行清除作业。

## 利用进程衍生进行内存管理

Resque worker使用fork系统调用进行内存管理。我们先来看看相关代码(Ruby)，然后逐行分析。

```ruby
  if @child = fork
    srand #Reseeding
    procline "Forked #{@child} at #{Time.now.to_i}"
    Process.wait(@child)
  else
    procline "Processing #{job.queue} since #{Time.now.to_i}"
    perform(job, &block)
    exit! unless @cant_fork
  end
```

Resque每清除一个作业都会执行这些代码。

先解释一下这里的if/else, fork系统调用允许运行中的进程以编程的形式创建新的进程，这个新进程和原始进程一模一样，我们把调用fork的进程称为父进程，新创建的进程为子进程。子进程从父进程处继承了其所占用内存中的所有内容，以及所有属于父进程的已打开的文件描述符。这里的执行顺序跟ruby的方法fork返回值有关。在父进程中，fork返回新创建的子进程的pid（进程的唯一标识），在子进程中，fork返回nil，所以一个为真，一个为假。因此if语句块中的代码是由父进程执行的，而else语句块中的代码是子进程执行的。

然后，我们从查看父进程中的代码开始。

```ruby
  srand #Reseeding
```
这一行的存在纯粹是因为MRI ruby的某个修正版bug。。。

```ruby
  procline "Forked #{@child} at #{Time.now.to_i}"
```
procline是Resque更新当前进程名的内部方式。

```ruby
  Process.wait(@child)
```
Process.wait是系统阻塞调用，ruby库帮我们实现了。变量@child是指fork调用的值。因此在父进程中会是子进程的pid。这行代码告诉父进程一直阻塞到子进程结束。代码很直观吧！

现在我们来看子进程中发生了什么。

```ruby
  procline "Processing #{job.queue} since #{Time.now.to_i}"
```
同样这也是给进程设置一个名称，不过这里时给子进程。

```ruby
  perform(job, &block)
```
在这个子进程中，作业由Resque来执行。

```ruby
  exit! unless @cant_fork
```
然后子进程退出。

## 何必自找麻烦？
Resque这么做并不是为了实现并发或提速。实际上，它为每个作业处理多添加了一个步骤，这使得整个处理过程更慢了。那干嘛要自找麻烦？为什么不逐一处理作业？

Resque使用fork系统调用来确保其工作进程使用的内存不会膨胀。让我们来看一下当Resque worker衍生出进程后发生了什么，这对Ruby虚拟机又回产生怎样的影响。

fork系统调用创建了一个和原始进程一模一样的新进程。在这种情况下，原始进程只是预先载入了应用程序环境而已。因此我们知道在fork之后，会获得一个载入了应用程序环境的新进程。

然后子进程执行清除作业的任务，这就是内存使用出现问题的地方。后台作业可能需要将图像文件载入主存进行处理，从数据库读入大量的ActiveRecord对象，或者执行其他消耗大量内存的操作。

一旦子进程处理完作业并退出，将会释放其所占用的全部内存并交由操作系统进行清理。然后原始进程就可以恢复运行，同样只载入应用程序环境。

就内存而言，每当Resque处理完一个作业，都会返回到洁净状态。也就是说，处理作业时，内存使用量或会激增，但终会回落到一个适宜的基线水平。

