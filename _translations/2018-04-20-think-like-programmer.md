---
layout: post
title: 如何像程序员一样思考
subtitle: How to think like a programmer
author: Richard Reis
tags: [Programmer]
---

&emsp;&emsp;如果你对编程感兴趣，你可能听过下面这句引言：
> “Everyone in this country should learn to program a computer, because it teaches you to think.” — Steve Jobs

&emsp;&emsp;你可能也想知道它到底是指什么意思，像程序员一样思考？应该怎么做？本质上讲，这就是解决问题更有效的方式。本文将为你讲述如何像程序员一样思考，读完这篇文章后，你将会确切地知道应该通过哪些步骤去解决问题。

### 为什么这很重要？
&emsp;&emsp;解决问题的能力是**元技能**。

&emsp;&emsp;我们都会遇到问题，大的或小的。有时候，我们如何解决这些问题其实是很随机的。除非你拥有一个系统的方案，否则你可能就像下面这样来“解决”问题的：
1. 尝试一个方案；
2. 如果不管用，再尝试另一个方案；
3. 重复第2步直到运气够好找到正确的方案...

&emsp;&emsp;看看，有时候运气好才能找到正确的解决方案。但这是最糟糕的解决问题的方式！而且非常非常浪费时间。解决问题最佳的方式应该包括两个步骤：
1. 系统的框架；
2. 练习实践；

> “Almost all employers prioritize problem-solving skills first.
> Problem-solving skills are almost unanimously the most important qualification that employers look for….more than programming languages proficiency, debugging, and system design.
>
> Demonstrating computational thinking or the ability to break down large, complex problems is just as valuable (if not more so) than the baseline technical skills required for a job.” — Hacker Rank (2018 Developer Skills Report)


### 系统的框架
&emsp;&emsp;为了找到正确的框架，我遵循了Tim Ferriss关于学习的书<<The 4-Hour Chef>>的建议，采访了两个非常出色的人：C.Jordan Ball（在Coderbyte的65,000多名用户中排名第一或第二）和 V. Anton Spraul（“像程序员一样思考：创造性解决问题介绍“的作者）。我问了他们相同的问题，你猜怎么着？他们的回答惊人的相似！很快你就会看到他们的答案。

&emsp;&emsp;这并不是意味着他们做所有事情都是一样的。每个人做事的方法都是不同的，你也会不同。但如果你从大家都认同的规则出发，你会走得更远更快。

> “The biggest mistake I see new programmers make is focusing on learning syntax instead of learning how to solve problems.” — V. Anton Spraul

&emsp;&emsp;所以，当你遇到新问题时你应该怎么做？这些是正确的步骤：

#### 1. 了解问题
&emsp;&emsp;确切地了解问题是什么。大部分难题之所以难是因为你根本不了解它们（这就是为什么了解问题是第一步的原因）。如何知道你是否了解一个问题？当你了解了一个问题时，你可以用简单的语言来描述它。你是否记得被某个问题难住的时候，当你开始尝试描述它时，立刻发现了之前没有注意到的逻辑漏洞？

&emsp;&emsp;大部分程序员都知道这种感觉。

&emsp;&emsp;这就是为什么你应该把你的问题写下来，画一张图表，或者把它告诉其他人（或物...有些人使用橡皮鸭）。

> “If you can’t explain something in simple terms, you don’t understand it.” — [Richard Feynman](https://en.wikipedia.org/wiki/Richard_Feynman)

#### 2. 计划方案
&emsp;&emsp;不要在没有计划的情况下直接去解决问题，计划好你的解决方案！如果你不能写下确切的步骤，没有什么可以帮助你。在编程中，这意味着不要马上开始黑客攻击，让你的大脑有时间去分析问题并处理信息。为了得到一个好的计划，你需要回答出这个问题：“给定输入X，返回输出Y所需的步骤是什么？”。程序员有一个伟大的工具来帮助他们...注释！

#### 3. 分解问题
&emsp;&emsp;注意，这是最重要的步骤。

&emsp;&emsp;不要尝试去解决一个很大的问题，你会哭的。相反，把它分解成多个子问题，这些子问题会更容易解决。然后，逐个地解决这些子问题。从最简单的问题开始，最简单意味着你已经知道答案了（或者接近答案）。解决子问题不需要依赖其他子问题，一旦你把所有子问题都解决了，把所有子问题的答案连接起来，你便得到了整个大问题的解决方案。

&emsp;&emsp;请记住，分解问题是解决问题的基石。（如果有必要，请再次阅读此步骤）

#### 4. 卡住了？
&emsp;&emsp;现在，你可能坐在那里想：“嘿，伙计...这真的非常酷，可是万一我被卡住了甚至连一个子问题都没有办法解决呢？”

&emsp;&emsp;首先，深呼吸一下；其次，这很公平。不要担心，朋友。这发生在每个人的身上！不同之处在于最好的程序员/问题解决者对bug/错误更加好奇而不是恼火。事实上，当你面对一个难题的时候你应该尝试下面三件事情：
* **调试**：一步一步跟踪你的解决方案，尝试找到是不是哪里出错了。程序员称之为－调试。
> “The art of debugging is figuring out what you really told your program to do rather than what you thought you told it to do.”” — Andrew Singer

* **重新评估**：退后一步，从另一个角度来看问题。是不是有些东西可以抽象成更通用的方法？重新评估的另一种方式是重新开始。删除所有内容，然后重新开始。我是认真的。你会惊讶于这是多么有效。
* **搜索**：Google。不管你遇到了什么问题，有人可能已经解决了它。找到那个人或者那个解决方案。事实上，即使你解决了问题，也要这么做（你可以从其他人的解决方案中学到很多东西）！警告：不要为大问题寻找解决方案，只为子问题寻找解决方案。为什么？因为一旦你陷入挣扎，你就不会学到任何东西，如果你没有学到任何东西，就是浪费了你的时间。

### 实践
&emsp;&emsp;不要指望一周后你就会很棒。如果你想成为一个好的问题解决者，那就去解决更多的问题。

&emsp;&emsp;实践，实践，实践，重要的事情说三遍。你意识到：“这个问题很容易用xxx来解决！”只是时间问题。如何实践？有很多选择，比如国际象棋谜题，数学问题，数独，围棋，大富翁，视频游戏等等。事实上，成功人士的共同模式是他们练习“微解决问题”的习惯。举个例子，[Peter Thiel](https://en.wikipedia.org/wiki/Peter_Thiel) 玩国际象棋，[Elon Musk](https://en.wikipedia.org/wiki/Elon_Musk) 玩视频游戏。这是否意味着你应该玩视频游戏？一点也不。但是为什么玩视频游戏呢？没错，解决问题！所以你应该做的是找到一个实践的出路，一个可以让你解决很多微观问题的东西（理想情况下，你喜欢的东西）。例如，我喜欢编码挑战。我每天都试着解决至少一项挑战（通常在[Coderbyte](https://coderbyte.com/)上）。

&emsp;&emsp;正如我所说，所有问题都有相似的模式。

### 结语
&emsp;&emsp;现在，你更清楚“像程序员一样思考”的含义。你也知道解决问题是在培养一种难以置信的技能（**元技能**）。另外，你也知道该怎么做才能练习解决问题的能力！

&emsp;&emsp;很酷吧？最后，我希望你遇到更多的问题^_^。

&emsp;&emsp;你没有看错，至少现在你知道如何解决它们！ 并且，你会学习每个解决方案，进而改进它们。

&emsp;&emsp;现在，去解决一些问题吧，good luck !



_[原文](https://medium.freecodecamp.org/how-to-think-like-a-programmer-lessons-in-problem-solving-d1d8bf1de7d2)，文章部分引用稍作修改，大部分依据译者理解来翻译，如有错漏，敬请谅解。_
