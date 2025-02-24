# 9.3 使用Zookeeper实现非扩展锁

这一部分我想讨论的例子是非扩展锁。我讨论它的原因并不是因为我强烈的认为这种锁是有用的，而是因为它在Zookeeper论文中出现了。

对于锁来说，常见的操作是Aquire Lock，获得锁。获得锁可以用下面的伪代码实现：

```
WHILE TRUE:
    IF CREATE("f", data, ephemeral=TRUE): RETURN
    IF EXIST("f", watch=TRUE):
        WAIT
```

在代码的第2行，是尝试创建锁文件。除了指定文件名，还指定了ephemeral为TRUE（ephemeral的含义详见9.1）。如果锁文件创建成功了，表明我们获得了锁，直接RETURN。

如果锁文件创建失败了，我们需要等待锁释放。因为如果锁文件创建失败了，那表明锁已经被别人占住了，所以我们需要等待锁释放。最终锁会以删除文件的形式释放，所以我们这里通过EXIST函数加上watch=TRUE，来监测文件的删除。在代码的第3行，可以预期锁文件还存在，因为如果不存在的话，在代码的第2行就返回了。

在代码的第4行，等待文件删除对应的watch通知。收到通知之后，再回到循环的最开始，从代码的第2行开始执行。

所以，总的来说，先是通过CREATE创建锁文件，或许可以直接成功。如果失败了，我们需要等待持有锁的客户端释放锁。通过Zookeeper的watch机制，我们会在锁文件删除的时候得到一个watch通知。收到通知之后，我们回到最开始，尝试重新创建锁文件，如果运气足够好，那么这次是能创建成功的。

在这里，我们要问自己一个问题：如果多个客户端并发的请求锁会发生什么？

有一件事情可以确定，如果有两个客户端同时要创建锁文件，Zookeeper Leader会以某种顺序一次只执行一个请求。所以，要么是我的客户端先创建了锁文件，要么是另一个客户端创建了锁文件。如果我的客户端先创建了锁文件，我们的CREATE调用会返回TRUE，这表示我们获得了锁，然后我们直接RETURN返回，而另一个客户端调用CREATE必然会收到了FALSE。如果另一个客户端先创建了文件，那么我的客户端调用CREATE必然会得到FALSE。不管哪种情况，锁文件都会被创建。当有多个客户端同时请求锁时，因为Zookeeper一次只执行一个请求，所以还好。

如果我的客户端调用CREATE返回了FALSE，那么我接下来需要调用EXIST，如果锁在代码的第2行和第3行之间释放了会怎样呢？这就是为什么在代码的第3行，EXIST前面要加一个IF，因为锁文件有可能在调用EXIST之前就释放了。如果在代码的第3行，锁文件不存在，那么EXIST返回FALSE，代码又回到循环的最开始，重新尝试获得锁。

类似的，并且同时也更有意思的是，如果正好在我调用EXIST的时候，或者在与我交互的副本还在处理EXIST的过程中，锁释放了会怎样？不管我与哪个副本进行交互，在它的Log中，可以确保写请求会以某种顺序执行。所以，与我交互的副本，它的Log以某种方式向前增加。因为我的EXIST请求是个只读请求，所以它必然会在两个写请求之间执行。现在某个客户端的DELETE请求要在某个位置被处理，所以，在副本Log中的某处是来自其他客户端的DELETE请求。而我的EXIST请求有两种可能：要么完全的在DELETE请求之前处理，这样的话副本会认为，锁文件还存在，副本会在WATCH表单（详见8.7）中增加一条记录，之后才执行DELETE请求。

![](<../.gitbook/assets/image (297).png>)

而当执行DELETE请求的时候，可以确保我的WATCH请求在副本的WATCH表单中，所以副本会给我发送一个通知，说锁文件被删除了。

要么我的EXIST请求在DELETE请求之后处理。这时，文件并不存在，EXIST返回FALSE，又回到了循环的最开始。

![](<../.gitbook/assets/image (298).png>)

因为Zookeeper的写请求是序列化的，而读请求必然在副本Log的两个写请求之间确定的位置执行，所以这种情况也还好。

> 学生提问：如果EXIST返回FALSE，回到循环最开始，调用CREATE的时候，已经有其他人创建了锁会怎样呢？
>
> Robert教授：那么CREATE会返回FALSE，我们又回到了EXIST，这次我们还是需要等待WATCH通知锁文件被删除了。

> 学生提问：为什么我们不关心锁的名字？
>
> Robert教授：这只是一个名字，为了让不同的客户端可以使用同一个锁。所以，它只是个名字而已。当我获得锁之后，我可以对锁保护的数据做任何操作。比如，一次只有一个人可以在这个课堂里讲课，为了讲课，首先需要获得这个课堂的锁，那要先知道锁的名字，比如说34100（猜是教室名字）。这里讨论的锁本质上就是一个znode，但是没有人关心它的内容是什么。所以，我们需要对锁有一个统一的名字。所以，Zookeeper看起来像是一个文件系统，实际上它是一个命名系统（naming system）。

这里的锁设计并不是一个好的设计，因为它和前一个计数器的例子都受羊群效应（Herd Effect）的影响。所谓的羊群效应，对于计数器的例子来说，就是当有1000个客户端同时需要增加计数器时，我们的复杂度是 $$O(n^2)$$ ，这是处理完1000个客户端的请求所需要的总时间。对于这一节的锁来说，也存在羊群效应，如果有1000个客户端同时要获得锁文件，为1000个客户端分发锁所需要的时间也是 $$O(n^2)$$ 。因为每一次锁文件的释放，所有剩下的客户端都会收到WATCH的通知，并且回到循环的开始，再次尝试创建锁文件。所以CREATE对应的RPC总数与1000的平方成正比。所以这一节的例子也受羊群效应的影响，像羊群一样的客户端都阻塞在Zookeeper这。这一节实现的锁有另一个名字：非扩展锁（Non-Scalable Lock）。它对应的问题是真实存在的，我们会在其他系统中再次看到。
