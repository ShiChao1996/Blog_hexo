---
title: 正则表达式入门
cover: https://tse3-mm.cn.bing.net/th?id=OIP.5HaX26pF6-WOmiKfHtTWkgHaEK&w=290&h=162&c=7&o=5&pid=1.7
author: 
  nick: Lovae
tags: 
    - regex 
    
date: 2017-12-17
categories: other

# 首页每篇文章的子标题
subtitle: regex 入门

---
## 原理

![](http://upload-images.jianshu.io/upload_images/2463524-dcbd871af2bc547d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2)

#### 正则引擎

>为什么正则能有效，因为有引擎，这和为什么JS能执行一样，有JS引擎

正则的引擎大致可分为两类：DFA和NFA

* DFA (Deterministic finite automaton) 确定型有穷自动机
* NFA (Non-deterministic finite automaton) 非确定型有穷自动机，大部分都是NFA

这里的“确定型”指，对于某个确定字符的输入，这台机器的状态会确定地从a跳到b，“非确定型”指，对于某个确定字符的输入，这台机器可能有好几种状态的跳法；这里的“有穷”指，状态是有限的，可以在有限的步数内确定某个字符串是被接受还是发好人卡的；这里的“自动机”，可以理解为，一旦这台机器的规则设定完成，就可以自行判断了，不要人看。

#### 基础知识

![](http://upload-images.jianshu.io/upload_images/2463524-8ddaddee044d96fd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>正则眼中的字符串——n个字符，n+1个位置

>为什么要有字符还要有位置呢？因为位置是可以被匹配的。

“占有字符”和“零宽度”:
* 如果一个子正则表达式匹配到的是字符，而不是位置，而且会被保存到最终的结果中，那个这个子表达式就是占有字符的，比如  /ha/  （匹配   ha  ）就是占有字符的；
* 如果一个子正则匹配的是位置，而不是字符，或者匹配到的内容不保存在结果中（其实也可以看做一个位置），那么这个子表达式是零宽度的，比如     /read(?=ing)/   （匹配     reading     ，但是只将read放入结果中，下文会详述语法，此处仅仅举例用），其中的(?=ing)就是零宽度的，它本质代表一个位置。

>占有字符是互斥的，零宽度是非互斥的。也就是一个字符，同一时间只能由一个子表达式匹配，而一个位置，却可以同时由多个零宽度的子表达式匹配。举个栗子，比如/aa/是匹配不了a的，这个字符串中的a只能由正则的第一个a字符匹配，而不能同时由第二个a匹配（废话）；但是位置是可以多个匹配的，比如/\b\ba/是可以匹配a的，虽然正则表达式里有2个表示单词开头位置的\b元字符，这两个\b是可以同时匹配位置0（在这个例子中）的


##### 控制权和传动

控制权是指哪一个正则子表达式（可能为一个普通字符、元字符或元字符序列组成）在匹配字符串，那么控制权就在哪。

传动是指正则引擎的一种机制，传动装置将定位正则从字符串的哪里开始匹配。

>正则表达式当开始匹配的时候，一般是由一个子表达式获取控制权，从字符串中的某一个位置开始尝试匹配，一个子表达式开始尝试匹配的位置，是从前一子表达匹配成功的结束位置开始的


## 语法
![](http://upload-images.jianshu.io/upload_images/2463524-62ea8063e4cf5821.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


####  要用某类常见字符——简单元字符

* '.' 匹配除了换行符以外的任意字符，也即是[^\n]，如果要包含任意字符，可使用(.|\n)
*  \w    (whatever) 匹配任意字母、数字或者下划线，等价于[a-zA-Z0-9_]，在deerchao的文中还指出可匹配汉字，但是\w在JS中是不能匹配汉字的
* \s     (space)  匹配任意空白符，包含换页符\f、换行符\n、回车符\r、水平制表符\t、垂直制表符\v
* \d    匹配数字
* \un   (Unicode) 匹配n，这里的n是一个有4个十六进制数字表示的Unicode字符，比如\u597d表示中文字符“好”，那么超过\uffff编号的字符怎么表示呢？ES6的u修饰符会帮你。

#### 要表示出现次数（重复）——限定符

* a*表示字符a连续出现次数 >= 0 次
* a+表示字符a连续出现次数 >= 1 次
* a?表示字符a出现次数 0 或 1 次
* a{5}表示字符a连续出现次数 5 次
* a{5,}表示字符a连续出现次数 >= 5次
* a{5,10}表示字符a连续出现次数为 5到10次 ，包括5和10

####  匹配位置——定位符和零宽断言

*   \b  匹配单词边界位置，准确的描述是它匹配一个位置，这个位置前后不全是\w能描述的字符，所以像\u597d\babc是可以匹配“好abc”的。
*   ^   匹配字符串开始位置，也就是位置0，如果设置了 RegExp 对象的 Multiline 属性，^ 也匹配 '\n' 或 '\r' 之后的位置
*   $   匹配字符串结束位置，如果设置了RegExp 对象的 Multiline 属性，$ 也匹配 '\n' 或 '\r' 之前的位置


#### 想表达“或”的意思——字符簇和分歧

>字符簇可用来表达字符级别的“或”语义，表示的是方括号中的字符任选一：

* [abc]表示a、b、c这3个字符中的任意一个，如果字母或者数字是连续的，那么可以用-连起来表示，[b-f]代表从b到f这么多字符中任选一个
* [(ab)(cd)]并不会用来匹配字符串“ab”或“cd”，而是匹配a、b、c、d、(、)这6个字符中的任一个，也就是想表达“匹配字符串ab或者cd”这样的需求不能这么做，要这么写ab|cd。但这里要匹配圆括号本身，讲道理是要反斜杠转义的，但是在方括号中，圆括号被当成普通字符看待，即便如此，仍然建议显式地转义
* 分歧用来表达表达式级别的“或”语义，表示的是匹配|左右任一表达就可：
ab|cd会匹配字符串“ab”或者“cd”
* 会短路，回想下编程语言中逻辑或的短路，所以用(ab|abc)去匹配字符串“abc”，结果会是“ab”，因为竖线左边的已经满足了，就用左边的匹配结果代表整个正则的结果


#### 想表达“非”的意思——反义

*   \W、\D、\S、\B 用大写字母的这几个元字符表示就是对应小写字母匹配内容的反义，这几个依次匹配“除了字母、数字、下划线外的字符”、“非数字字符”、“非空白符”、“非单词边界位置”
* [^aeiou]表示除了a、e、i、o、u外的任一字符，在方括号中且出现在开头位置的^表示排除，如果^在方括号中不出现在开头位置，那么它仅仅代表^字符本身

#### 贪婪和非贪婪

在限定符中，除了{n}确切表示重复几次，其余的都是一个有下限的范围。

在默认的模式（贪婪）下，会尽可能多的匹配内容。比如用ab*去匹配字符串“abbb”，结果是“abbb”。

而通过在限定符后面加问号?可以进行非贪婪匹配，会尽可能少地匹配。用ab*?去匹配“abbb”，结果会是“a”。

不带问号的限定符也称匹配优先量词，带问号的限定符也称忽略匹配优先量词。

## JS 中的正则

>字面量, 构造函数和工厂符号都是可以的：

             /pattern/flags
             new RegExp(pattern [, flags])
             RegExp(pattern [, flags])


参数
pattern
正则表达式的文本。
flags
如果指定，标志可以具有以下值的任意组合：

* g 全局匹配;找到所有匹配，而不是在第一个匹配后停止

* i 忽略大小写

* m 多行; 将开始和结束字符（^和$）视为在多行上工作（例如，分别匹配每一行的开始和结束（由 \n 或 \r 分割），而不只是只匹配整个输入字符串的最开始和最末尾处。

* u Unicode; 将模式视为Unicode序列点的序列

* y 粘性匹配; 仅匹配目标字符串中此正则表达式的lastIndex属性指示的索引(并且不尝试从任何后续的索引匹配)。


>有两种方法来创建一个RegExp对象：一是字面量、二是构造函数。要指示字符串，字面量的参数不使用引号，而构造函数的参数使用引号。因此，以下表达式创建相同的正则表达式


            /ab+c/i;
            new RegExp('ab+c', 'i');
            new RegExp(/ab+c/, 'i');

#### 方法

>RegExp.prototype.exec()

exec() 方法在一个指定字符串中执行一个搜索匹配。返回一个结果数组或 null。

            var matches = /h./.exec('This is a hello world!');
            console.log(matches);        // [ 'hi', index: 1, input: 'This is a hello world!' ]

>RegExp.prototype.test()

test() 方法执行一个检索，用来查看正则表达式与指定的字符串是否匹配。返回 true 或 false。

当你想要知道一个模式是否存在于一个字符串中时，就可以使用 test()（类似于 String.prototype.search() 方法），差别在于test返回一个布尔值，而 search 返回索引（如果找到）或者-1（如果没找到）

            let str = 'hello world!';
            let result = /hello/.test(str);
            console.log(result);
            // true

## 例子
>使用正则改变数据结构

下例使用 replace 方法 （继承自 String）去匹配姓名 first last 输出新的格式 last, first。脚本中使用 $1 和 $2 指明括号里先前的匹配.

            var re = /(\w+)\s(\w+)/;
            var str = "John Smith";
            var newstr = str.replace(re, "$2, $1");
            print(newstr);          //"Smith, John"


>在多行中使用正则表达式

            var s = "Please yes\nmake my day!";
            s.match(/yes.*day/);
            // Returns null
            s.match(/yes[^]*day/);
            // Returns 'yes\nmake my day'

>使用正则表达式和 Unicode 字符

\w 或 \W 只会匹配基本的 ASCII 字符；如 'a' 到 'z'、 'A' 到 'Z'、 0 到 9 及 '_'。为了匹配其他语言中的字符，如西里尔（Cyrillic）或 希伯来语（Hebrew），要使用 \uhhhh，"hhhh" 表示以十六进制表示的字符的 Unicode 值

            var text = "Образец text на русском языке";
            var regex = /[\u0400-\u04FF]+/g;

            var match = regex.exec(text);
            print(match[1]);  // prints "Образец"
            print(regex.lastIndex);  // prints "7"

            var match2 = regex.exec(text);
            print(match2[1]);  // prints "на" [did not print "text"]
            print(regex.lastIndex);  // prints "15"

            // and so on

>从 URL 中提取子域名

            var url = "http://xxx.domain.com";
            print(/[^.]+/.exec(url)[0].substr(7)); // prints "xxx"




