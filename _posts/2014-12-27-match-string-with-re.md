---
layout: post
title:  "Match C style String with RE"
date:   2014-11-26 00:54:45
categories: jekyll update
---
# 使用正则表达式匹配带转义字符的字符串

最近写一个东西, 处理输入的命令, 需要进行简单的词法分析, 其中比较麻烦的是字符串.


比如


    echo "Just a test.\"OK\"."


这句命令包含两个字符串, 前面这个不用说了. 关键是第二个, 是一个C风格的字符串, 如果手写解析当然很容易, 但是如果能够直接用正则表达式进行匹配, 就可以直接利用flex/bison等现成的工具了.


从简单的开始, 先不考虑有转义字符的存在, 那么首先想到的正则表达式是

    ".*"

但是试着匹配一下会发现其实不对

    >>> re.match(r'".*"', '"abcdefg"hijklmn"')
    <_sre.SRE_Match object; span=(0, 17), match='"abcdefg"hijklmn"'>

因为`*`是最大匹配, 也就是匹配的时候会匹配尽量多的字符, 如果后面还有`"`, 会继续匹配下去. 修改为最小匹配即可

    ".*?"

测试下

    >>> re.match(r'".*?"', '"abcdefg"hijklmn"')
    <_sre.SRE_Match object; span=(0, 9), match='"abcdefg"'>

Good. 接下来考虑带转义字符的情况, 比如

    "This is a book.\nIts name is \"The One.\""

先测试一下用之前的表达式匹配会是什么结果, 按理说应该匹配`This is a book.\nIts name is \"`

    >>> re.match(r'".*?"', '"This is a book.\nIts name is \\"The One.\\""')
    >>>

咦? 看样子是`\n`中断了匹配, 加个flag

    >>> re.match(r'".*?"', '"This is a book.\nIts name is \\"The One.\\""', re.DOTALL)
    <_sre.SRE_Match object; span=(0, 31), match='"This is a book.\nIts name is \\"'>

和我们预想的结果一样. 由于`\`是我们自定义的转义字符, 正则表达式根本就不会因为他的存在而忽略掉后面的`"`.

仔细考察一下字符串的特征, `\n`, `\r`等等都不会影响匹配, 因为没有出现`"`. `"`不中断匹配的条件就是被转义, 也就是`\"`, 考虑到`\`本身也会被转义, 所以`\\"`虽然带有`\"`, 但是也是会中断匹配的. 如此归纳一下, 也就是`\(\\)*"`(奇数个`\`)不中断匹配, `(\\)*"`(偶数个`\`)中断匹配.

所以, 我们需要的正则表达式是

    ".*?(\\)*?"

恩哼! 很容易嘛, 赶紧试试

    >>> re.match(r'".*?(\\)*?"', '"This is a book.\nIts name is \\"The One.\\\\""', re.DOTALL)
    <_sre.SRE_Match object; span=(0, 31), match='"This is a book.\nIts name is \\"'>

靠...不对. 为什么呢?

`*`表示0次以上的重复, 也就是其实`\"`也是可以被匹配上的... 那么如何排除这种情况呢? 在手册里查了一下, 发现`(?<!...)`, 用来排除前置的情况刚刚好!

    ".*?(?<!\)(\\)*?"

赶快试试!!(咆哮)

    >>> re.match(r'".*?(?<!\\)(\\\\)*?"', '"This is a book.\nIts name is \\"The One.\\""', re.DOTALL)
    <_sre.SRE_Match object; span=(0, 42), match='"This is a book.\nIts name is \\"The One.\\""'>

哈哈哈哈哈~~~~

最后说一句题外话, 如果去掉开头和结尾的`"`, 是没有办法正确匹配的, 必须将`*?`替换为`*`, 想想为什么?

END
