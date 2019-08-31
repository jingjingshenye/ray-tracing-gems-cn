# 序言

光线追踪现已成为实时渲染中的核心组成部分。我们目前使用着GPU及其API来加速光线跟踪。但我们也需要算法，高质量的算法能让程序以每秒60帧画面甚至更快的速度执行，与此同时，还能在每帧带来高质量的图像。这些算法正是本书所要阐述的。

人们通常会很容易忽略序言而直奔正文，但我们还是想确保让你知道两件事情：

1. 你可以在这个链接中找到与本书相关的补充代码和资料：http://raytracinggems.com
2. 本书的所有内容都可以公开获取。

第二条听起来似乎没那么让人激动，但这意味着只要你能信守承诺，保证不用于商业目的，你就可以自由地复制和分享任何章节或是整本书的内容。本书的具体许可遵循协定Creative Commons Attribution 4.0 International License (CC-BY-NC-ND)，https://creativecommons.org/licenses/by-nc-nd/4.0/ 我们把这些内容公布于此，目的就是让作者和其他所有人都可以尽快地分享资源。

这里要依次感谢你们，你们每位参与者的倾力支持，是整个编撰项目中最令人愉悦的事情之一。我们在NVIDIA找到了Aaron Lefohn、David Luebke、Steven Parker和Bill Dally，当我们表达了想要撰写一本类似Gems风格讲述光线追踪的书时，他们当即表示，若能付诸实现，那真的是太棒了。在他们的帮助下，项目已经完成了。在此，我们要感谢他们。

我们非常感谢Anthony Cascio和Nadeem Mohammad对网站以及项目提交系统的支持。与此同时，我们还要特别感谢Nadeem所做的合同谈判相关事宜，这确保了所有人都可以完全开放地查阅本书，并能免费获取本书的电子版。

这本书的编撰排期非常紧张，如果没有NVIDIA创意团队和Apress出版商制作团队的大力奉献，本书的出版将会延期甚久。创意团队中的许多成员参与了将短片《Project Sol》生成影像照片的工作。本书封面以及七大部分的起始页配上了照片之后，效果精美绝伦。我们要特别感谢Amanda Lam、Rory Loeb和T.J. Morales，他们为本书的所有图表字符设计了统一的风格，同时还提供了本书的封面设计和每章节引言部分的排版。我们还要感谢NVIDIA的Dawn Bardon、Nicole Diep、Doug MacMillan和Will Ramey，他们为我们提供了后勤保障工作。

我们对Natalie Pao和Apress制作团队充满了感激之情。他们不知疲倦地和我们一起工作，在最后截稿的时间里，一起解决了很多问题。

另外，我们要感谢以下人员，他们付出了额外的心血，为的就是让本书内容更加精彩。他们是：Pontus Andersson、Andrew Draudt、Aaron Knoll、Brandon Lloyd和Adam Marrs。

本书能成功出版，在很大程度上要归功于我们编辑部的梦幻组合：Alexander Keller、Morgan McGuire、Jacob Munkberg、Matt Pharr、Peter Shirley、 Ingo Wald和Chris Wyman。他们认真仔细地审稿、一丝不苟地修订，还在必要之时另找审阅人参与编辑。

最后，本书的出版离不开每个章节的作者，他们无私地向图形学社区传授知识、分享经验。他们用尽了各种方法来润色章节内容，常常是在几小时甚至是几分钟之内，要求我们再做一次修改、确认或计算。感谢你们所有人！

<p align="right">——Eric Haines和Tomas Akenine-Möller</p>
<p align="right">2019年1月</p>
