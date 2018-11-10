---
layout: post
title:  "Paxos-simple-simple——一定让你懂Paxos"
date:   2018-10-28 14:06:05
categories: 服务端
tags: 服务端 Web C++ Java
---
* content
{:toc}

本文建议阅读时间为一个无人打扰的下午加上没人想你的晚上

## 1. 分布式一致性
我看过介绍自己公司架构演进的书，以及了解过一些大型互联网公司如google、facebook、的架构演进过程。所有的变化遵循着这个方向：<br/>
    单机部署->高质量小型机->分布式系统<br/>
既然最终会演进到分布式系统，那么自然会面临一些分布式特有的问题。
首当其冲，如何保证数据一致性，或者说最终一致性的问题。
在引入现成的工具，例如Zookeeper之前，有必要先了解，怎样的方式，能够实现数据一致性。<br/>
Lamport在他的论文[The Part-time Paliament](https://www.microsoft.com/en-us/research/uploads/prod/2016/12/The-Part-Time-Parliament.pdf)
中给出了算法Paxos，11年后，他又给出了另一篇论文，[Paxos-made-simple](http://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
用来给那些看不懂上一篇论文的人一个简单些的选择。<br/>
可惜，Paxos-made-simple对于大部分人来讲，仍然不是一遍就能理解paxos算法的文章。我先看过《从Paxos到ZooKeeper》，然后再看Paxos-made-simple原文，才算理解此算法。<br/>
这篇文章接下来，致力于剥离出Paxos-made-simple原文中的推导细节和证明细节，让你一遍懂Paxos。

## 2. 想要的结果{#ds}
分布式系统存在多个节点，这些节点可能是一台硬件机器，也可能是一个进程，我们想要这些节点各自提出数据提案，但最终的结果<strong id="result">result</strong>为：<br/>
1. 这些被提出的提案中，只有一个被选定。
2. 无提案被提出，则无提案被选中。
3. 当一个提案被选定后，节点们都可获取此提案的信息。

对于一致性来讲，一致性安全需求<strong id="safety">safety</strong>为:<br/>
1. 只有被提出的提案才能被选定
2. 只能有一个值被选定
3. 如果一个节点认为一个被选定，则此提案真的被选定。

并且，我们节点之间的通信，通信环境<strong id="environment">environment</strong>为：<br/>
1. 每个节点执行速度任意，可能出错而停止，也可能重启，同时，当一个提案被选定后，所有的参与者都有可能失败或重启，因此，除非那些失败或重启的参与者可记录某些信息，否则无法确定最终一致性。
2. 消息传输过程可能会延迟，也可能重复、丢失，但是消息不会被损坏，即消息内容不会被篡改。

所以，我们想要的是，在<a href="#environment">environment</a>下，达成<a href="#result">result</a>，同时满足<a href="#safety">safety</a>。

## 3. 节点行为的抽象
        
为达成这个结果，考虑，节点本身会进行哪些行为。我们对节点行为本身进行分析后，对这些行为进行约束，让能满足我们想要结果的做法通过，无法满足我们想要结果的行为屏蔽，即可找到此一致性算法。<br/>
对节点行为进行分析后，节点的行为可以分类为：<br/>
1. 发出提案请求
2. 接收提案请求，并选择是否批准提案请求 (批准提案是单个节点的行为，选定提案是整个分布式系统的行为)
3. 选定提案
4. 获取被选定的提案



对于行为3、选定一个提案，如果我们认为，当大多数节点都批准了同样的提案，那么，此提案被选定。
<br/>
<br/>
<strong id="most-rule-pre">most-rule</strong><br/>
- 如果大多数节点批准了同样的提案，那么此提案被选定。

为何数量上需要是大多数节点，因为任意两个大多数节点的集合，一定存在交集，这个交集中的节点，可在不同的提案中进行权衡，从而不同提案达成一致。<br/>
对于中国人来讲，可记忆为：少数服从多数。
由于对most-rule的认可，行为3不再是整个分布式系统的一个行为，它只是行为2到达一定节点数量后，产生的一个作用。
于是，在most-rule之下，节点的行为只剩下：1、2、4 <br/>
对此三种行为，进行抽象，将此三种行为封装为三种角色：<br/>
1. Proposer，对应于行为1，发出提案者
2. Acceptor，对应于行为2，接收提案者
3. Learner，对应于行为4，获取提案

由于此三种角色是对节点行为的抽象，故单一节点可能不止扮演一种角色。<br/>
现在，对于most-rule，可描述为：<br/>
    如果大多数的Acceptor批准了一个提案，此提案被选定。

## 4. 推导环节，考量节点行为的约束。

现在，我们可以考量，我们抽象出来的三种角色，Proposer，Acceptor，Learner，进行他们的行为时，需要满足哪些约束。
<br/>
<br/>
<strong id="hope">hope</strong>：为了产生<a href="#result">result</a>,我们希望，哪怕只有一个提案被提出，也能达成result，选定一个提案。
<br/>
<br/>
此种希望，谕示着:<br/>
<br/>
<strong id="p1">P1</strong>: 一个Accpetor必须批准它收到的第一个提案。
<br/>
<br/>
显然，如果Accpetor收到第一个提案后，要去斟酌权衡:"我觉得下一个提案或许会更好"，那么对于分布式系统而言，有可能永远没有下一个提案，此与我们的hope相悖。<br/>
ok,我们获得第一个对Acceptor的行为约束，P1。<br/>
但是，P1同时带来了一个问题，如果Acceptor无脑同意第一个提案，那么可能存在的情形是：同一时间，有不同的提案被提出，但是没有任何一个提案占据了大多数的Acceptor，于是没有一个提案被选定。(极端情形，两个不同的提案分别被一半一半的Acceptor所批准了)<br/>
如果,Acceptor能够批准不止一个提案，那么就可以同时满足<a href="#p1">P1</a>和<a href="#most-rule">most-rule</a>了。<br/>
即使在第一轮提案批准中，的确没有一个提案被大多数Acceptor接收到，即无法满足most-rule，无提案被选定。但是Acceptor可以继续接受并批准提案，这场分布式系统内部的通信会议可以继续下去。<br/>
现在，由于Acceptor可以批准多个提案，我们先对提案进行一个记录，让提案有编号，从而知晓提案被提出和同意的先后次序。于是，一个提案可以被表示为“[编号，Value]”。<br/>
对于提案的编号，可以使用开源的snowflake算法来生成。<br/>
当然，由于Acceptor可以批准多个提案，所以可能会有多个提案都满足了most-rule,于是多个提案被选定了。对于
<a href="#safety">safety</a>中第二条来讲，如果被选定的提案有多个，那么需要保证多个提案为同一个值。<br/>
幸运的是，我们现在对提案进行了编号，所以可以通过编号次序来满足这一点:
<br/>
<br/>
<strong id="p2">P2</strong>: 如果编号为M<sub>0</sub>、Value值为V<sub>0</sub>的提案被选定，那么所有比编号M0更高的，且被选定的提案，其Value也必须是V<sub>0</sub>。
<br/>
<br/>
为了满足P2，实际操作上，我们在提案被选定的入口，即Acceptor批准提案时，来做约束。即满足:
<br/>
<br/>
<strong id="p2a">P2a</strong>: 如果编号为M<sub>0</sub>、Value为V<sub>0</sub>的提案，那么所有比编号M<sub>0</sub>更高的，且被Acceptor批准的提案，其值也必须是V<sub>0</sub>。
<br/>
<br/>
到现在为止，得出Acceptor行为必须满足的约束有: <a href="#p1">P1</a>和<a href="#p2a">P2a</a>
然而P1和P2a存在逻辑上的矛盾，假设一个Acceptor从未接受过提案，但此时有个提案已经被大多数的Acceptor批准，所以这个提案被选定了，但另一个值不同的提案发送给从未接受过提案的Acceptor，那么根据P1规则，Acceptor必须批准这个提案，但是根据规则P2a，Acceptor不得批准这个提案。
<br/>
解决这个矛盾的办法就是当提案被选定后，不再产生值不同的提案，即对P2a进行条件强化:
<br/>
<br/>
<strong id="P2b">P2b</strong>: 如果一个提案M<sub>0</sub>的Value值V<sub>0</sub>被选定，那么所有的Proposer产生的编号更大的提案，其值都为V<sub>0</sub>。
<br/>
<br/>
P2b与P1两个约束之间可以相处融洽。但是对于编程来讲，约束P1对于Acceptor可以轻松实现，约束P2b却没有明确的how to do。
<br/>
对于P2b的约束，我们需要再进一步，找到一个可行性好的约束。
<br/>
思考P2b约束的执行，最终的执行结果为:
<br/>
如果某个提案[M<sub>0</sub>,V<sub>0</sub>]已经被选定了，提案M<sub>n</sub>的值Value为V<sub>0</sub>
<br/>
如果含有这样一个中间过程:
<br/>
假设编号在M<sub>0</sub>到M<sub>n-1</sub>之间的提案，其值都是V<sub>0</sub>。
<br/>
那么我们可以很容易通过第二数学归纳法来证明，P2b的执行结果是可以达到的。
<br/>
继续思考这个中间过程的达成时的结果：
<br/>
1. 首先，此时M<sub>0</sub>被选定，所以存在一个大多数的Acceptor集合C，C中的每个Acceptor都批准了提案M<sub>0</sub>
2. 集合C中的Acceptor，都批准了一个编号在M<sub>0</sub>到M<sub>n-1</sub>范围内的提案，并且每个编号在M<sub>0</sub>到M<sub>n-1</sub>范围内被Acceptor批准的提案，Value都为M<sub>0</sub>

为了达成这个“中间过程”结果，Proposer在提出提案M<sub>n</sub>时，可以去访问一个包含大多数Acceptor的集合S，由于集合C和S都包含了Acceptor中的大多数，那么被集合C批准，从而选定的提案M<sub>0</sub>，必然在集合S中存在至少一个Acceptor批准了提案[M<sub>0</sub>,V<sub>0</sub>]。Proposer通过对集合S的访问，即可知道自身的提案M<sub>n</sub>是否应当被提出。
<br/>
这个Proposer向集合S询问自身提案是否应该被提出的过程，可总结为约束:
<br/>
<br/>
<strong id="p2c">P2c</strong>: 对于任意的M<sub>n</sub>和V<sub>n</sub>，如果提案[M<sub>n</sub>,V<sub>n</sub>]被提出，那么肯定存在一个由半数以上的Acceptor组成的集合S，满足以下两个条件中的任意一个:
- S中不存在任何批准过编号小于Mn的提案的Acceptor。
- 选取S中所有Acceptor批准的编号小于M<sub>n</sub>的提案，其中编号最大的那个提案Value为V<sub>n</sub>。

<br/>
如果不满足P2c中的条件，那么Proposer不应该提出这个提案。也就是P2c约束如果被满足，那么可以通过第二数学归纳法证明得出P2b，P2b得到执行可保证P2a，P2a又可保证P2，P2与P1一起，可保证得到我们希望的一致性结果result和hope。
<br/>
逻辑过程为:

<a href="#p2c">P2c</a>=><a href="#p2b">P2b</a>=><a href="#p2a">P2a</a>=><a href="#p2">P2</a>;

<a href="#p2">P2</a>+<a href="#p1">P1</a>=<a href="#result">result</a>+<a href="#hope">hope</a>;

## 5. Proposer提交提案
有了P2c，我们可以总结出来Proposer的提案提交流程。

首先，Proposer会先生成提案，即先进行Prepare请求，以期满足P2c：

1. Proposer选择一个新的提案号M<sub>n</sub>，向一个包含大多数Acceptor的集合发送请求，要求如下<span id="response">回应</span>:
    - 如果Acceptor已经批准提案，那么就向Proposer反馈当前该Acceptor已经批准过的编号小于M<sub>n</sub>但为最大编号的那个提案的值。
    - 向Proposer承诺，不再批准任何编号小于M<sub>n</sub>的提案。（这条承诺是必要的，因为Acceptor已经向Proposer发送了比M<sub>n</sub>编号小的提案的值，如果再批准比Mn编号小的提案，这种行为是无法通知到已经收到响应的Proposer的，且与上一条回应逻辑相悖）
2. 如果Proposer收到了来自半数以上的Acceptor的响应，那么它可以产生提案[M<sub>n</sub>,V<sub>n</sub>],这里的V<sub>n</sub>是所有响应中编号为最大的提案的Value值。如果半数以上的Acceptor都没有同意任何提案，那么Proposer可以任意选择值V<sub>n</sub>。
<br/>                                                                                         

生成提案成功后，Proposer会将生成的提案再次发送给包含大多数Acceptor的集合，并期望获得它的批准。此即Accept请求。

## 6. Acceptor批准提案

由Proposer提交提案流程可知，Acceptor会收到两种类型的请求: Prepare和Accept。对于两种请求，Acceptor的响应策略应该为:
- <strong>Prepare请求</strong>： Acceptor在任何时候响应此类型请求，并满足<a href="#response">回应</a>。
- <strong>Accept请求</strong>: 在不违背Accept现有承诺的前提下，可以响应Accept请求。

不违背现有承诺,即不批准任何编号小于已经接受过的Prepare请求，加上约束<a href="#p1">P1</a>,可总结为:
<br/>
<br/>
<strong id="p1a">P1a</strong>: 一个Acceptor只要尚未响应过任何大于M<sub>n</sub>的Prepare请求，那么它就可以接受这个编号为M<sub>n</sub>的提案。
<br/>
<br/>
P1a包含了P1，但我们还可对其进行一定层度的优化。我们的运行环境<a href="#environment">environment</a>提到，消息可能会延迟，于是Acceptor可能会接收到比已经接收到的Prepare请求的M<sub>n</sub>提案，编号更小的的提案的Prepare请求。同样，由于消息可能会重复，于是Acceptor可能会收到它已经批准过的提案的Prepare请求。
<br/>
可对Acceptor行为做优化，优化为:
<br/>
- 若Acceptor已经响应了一个编号为M<sub>n</sub>的Prepare请求，那么对于任何编号小于M<sub>n</sub>的Prepare请求，Acceptor也不再回应。
- Acceptor可忽略它已经批准过的提案的Prepare请求。

此种优化策略下，Acceptor只需要存储：

<strong id="storage">storage</strong>
</be>
- 它所批准的过的最高提案号的提案
- 所响应过的Prepare请求中，最高的提案编号

即可满足它执行算法过程的上下文。并且哪怕Acceptor失败或者重启，依然能满足P2c。
<br/>
提到重启或失败，Prepare节点只要保证不会产生具有相同编号的提案，那么它可以丢弃任意的提案以及它所有的运行时状态。因为条件P2c由Acceptor存储上下文来满足。
<br/>
另外，由于Proposer可以丢弃任意提案，所以当它在试图生成一个更大编号的提案时，丢弃旧有提案是可在保持一致性的前提下的一个好选择。
<br/>
并且，如果Proposer知道Acceptor收到过的最大Prepare的提案编号多少，那么Proposer的提案丢弃更有效率。
<br/>
于是将Acceptor的行为继续优化为:
- Acceptor在任何时候响应一个编号为Mn的Prepare请求
    1. 若此编号大于它响应过的所有Prepare请求，那么Acceptor将它批准过的最大编号的提案回应过去。(回复:yes,[M<sub>z</sub>,V<sub>z</sub>]。 其中z<n)
    2. 若此编号小于它所响应的Prepare请求，那么Acceptor回应它所响应过的Prepare请求中编号最大的那个。(回复：no，M<sub>x</sub>。 其中x>n)
    

## 7. 提案选定过程陈述<span id="statement"></span>

结合Proposer的发送策略，以及Acceptor的处理策略，可得到一个两阶段的执行过程

<strong id="phase1">Phase1</strong>
<br/>
1. Proposer选择一个提案编号M<sub>n</sub>，然后向包含大多数Acceptor的一个集合发送编号为M<sub>n</sub>的Prepare请求。
2. 如果一个Acceptor收到一个编号为M<sub>n</sub>的Prepare请求
    - 若编号M<sub>n</sub>大于该Acceptor已经响应过的所有Prepare请求的编号，那么它将它已经批准过的最大编号的提案作为响应反馈给Proposer，同时该Acceptor会承诺不会再批准任何编号小于M<sub>n</sub>的提案。
    - 若编号M<sub>n</sub>小于该Acceptor已经响应过的Prepare请求，则Acceptor回应已经响应过的Prepare请求中，最大的提案编号。


<strong id="phase2">Phase2</strong>

1. 如果Proposer收到来自半数以上的Acceptor对于其发出的编号为M<sub>n</sub>的prepare请求的响应，那么它就将生成的提案[M<sub>n</sub>,V<sub>n</sub>]的Accept请求发送给一个任意的包含大多数Acceptor的集合。
2. 如果Acceptor收到[M<sub>n</sub>,V<sub>n</sub>]这个提案的Accept请求，只要该Acceptor尚未对编号大于M<sub>n</sub>的Prepare请求做出响应，它就可以通过这个提案。

## 8. Learner获取提案

Proposer和Acceptor通过上文的协作，负责了提案的选定。Learner角色则负责提案的获取。
<br/>
想要Learner获取一个已经被选定的前提是，该提案已经被半数以上的Acceptor批准。因此，最简单的做法就是:
- 一旦Acceptor批准了一个提案，就马上同步到所有的Learner。

但当一个提案被Learner获知时，至少会带来超过半数的Acceptor乘以Learner数量的通信，效率严重浪费。
<br/>
更好一点的做法是:
- 选取一个主Learner，Acceptor将提案被批准的消息同步到主Learner，由主Learner来负责通知其余Learner。

虽然需要其他Learner需要主Leaner另行通知才能获知被选定的方案，但是通信次数大大降低了。
<br/>
唯一担心的是，当主Learner节点故障后，状态丢失，并且服务不可用。
<br/>
既然主Learner不可发生故障，那么可以考虑多布置几个主Learner，于是引入一个更一般的方法:
- 选定一个主Learner集合，Acceptor将提案被批准的消息同步到此集合中，此集合中任何一个Leaner都可通知其余Learner。

另外，注意我们的<a href="#environment">environment</a>,Acceptor发送给Learner的消息可能会丢失，这可能会导致Learner无法获知一个已经被选定的提案。
<br/>
解决办法是:
<br/>
<br/>
<strong id="FACE/OFF">FACE/OFF</strong> (电影《变脸》英文名)
<br/>
- Learner作为一个Proposer，发送一个Prepare请求给Acceptor，这样Acceptor会在Prepare编号更高时，将已经批准过的提案发回给Learner。

## 9. 优化Proposer，提高活性。

Paxos算法的核心逻辑已经描述结束，但对于Proposer，需要预防一种类似"死锁"的情形。此情形为:
<br/>
<br/>
<strong id="circle-scenario">Circle-Scenario
<br/>
有两个Proposer按顺序产生一系列提案，当Proposer P1提出了一个提案M<sub>1</sub>，并且完成<a href="#phase1">阶段一</a>，此时Proposer P2提出提案M<sub>2</sub> (明显2>1),M<sub>2</sub>也完成了<a href="#phase1">阶段一</a>的提交，此时，提案M<sub>1</sub>后续的Accept请求才发出，于是被M<sub>2</sub>的Prepare请求拒绝，然后Proposer P1一怒之下，发出提案M<sub>3</sub>，提案M<sub>3</sub>也完成了<a href="#phase1">阶段一</a>,于是后续M<sub>2</sub>的Accept请求被拒绝。Proposer P2一怒之下，发出提案M<sub>4</sub> …… 
<br/>
<br/>
为了避免陷入Circle-Scenario，可以选择一个主Proposer，只要主Proposer才可以提出提案。这样一来，只要主Proposer能够与过半的Acceptor通信，那么整套算法就能保持活性。

## 10. 实现


Paxos算法假定有一个进程网络，在共识算法中，每一个进程扮演Proposer，Acceptor，Learner这些角色。算法选出一个leader进程，扮演主Proposer和主Learner。
<br/>
共识算法即上文所述,

好难翻译，待续................







