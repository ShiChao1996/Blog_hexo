---
title: dart 语法基础
cover: https://image.littlechao.top/20180224115242000010.jpg
author: 
  nick: Lovae
tags: 
    - dart
date: 2017-12-15
categories: dart

# 首页每篇文章的子标题
subtitle: dart 语法入门

---

>本文是对 Dart 语言的官方文档做了简单的翻译和总结，有不当之处敬请指正。
如果有时间和精力建议通读[官方文档](https://www.dartlang.org/guides/language/language-tour)

### hello world

            // Define a function.
            printNumber(num aNumber) {
              print('The number is $aNumber.'); // Print to console.
            }

            // This is where the app starts executing.
            main() {
              var number = 42; // Declare and initialize a variable.
              printNumber(number); // Call a function.
            }

----
### 重要的概念

* 能赋值给变量的所以东西都是对象，包括 numbers, null, function, 都是继承自 Object 内置类
* 尽量给变量定义一个类型，会更安全，没有显示定义类型的变量在 debug 模式下会类型会是  dynamic（动态的）
* dart 在 running 之前解析你的所有代码，指定数据类型和编译时的常量，可以提高运行速度
* dart 提供了顶级函数(如：main())
* dart 没有 public、private、protected 这些关键字，变量名以"_"开头意味着对它的 lib 是私有的

---
### 变量
>没有初始化的变量都会被赋予默认值 null

            var name = 'Bob';
            var unInitializeValue1;   //未给初值的变量，默认值为 null
            Int unInitializeValue2;   //即使是Int 型，默认值也是 null

程序中只当数据类型是为了指出自己的使用意图，并帮助语言进行语法检查。但是，指定类型不是必须的
###### num
* int 取值范围：-2^53 to 2^53

            // String -> int
            var one = int.parse('1');

            // String -> double
            var onePointOne = double.parse('1.1');

            // int -> String
            String oneAsString = 1.toString();

            // double -> String 注意括号中要有小数点位数，否则报错
            String piAsString = 3.14159.toStringAsFixed(2);

###### string
* '''...'''，"""..."""表示多行字符串
* r'...',r"..."表示“raw”字符串
* 用 $ 或 ${} 来计算字符串中变量的值

###### bool
* Dart 是强 bool 类型检查，**只有bool 类型的值是true 才被认为是true**

###### list
> list 基本和 JavaScript 数组一样，它的方法如下：

            // 使用List的构造函数，也可以添加int参数，表示List固定长度，不能进行添加 删除操作
            var vegetables = new List();

            // 或者简单的用List来赋值
            var fruits = ['apples', 'oranges'];

            // 添加元素
            fruits.add('kiwis');

            // 添加多个元素
            fruits.addAll(['grapes', 'bananas']);

            // 获取第一个元素
            fruits.first;

            // 获取元素最后一个元素
            fruits.last;

            // 查找某个元素的索引号
            assert(fruits.indexOf('apples') == 0);

            // 删除指定位置的元素，返回删除的元素
            fruits.removeAt(index);

            // 删除指定元素,成功返回true，失败返回false
            fruits.remove('apples');

            // 删除最后一个元素，返回删除的元素
            fruits.removeLast();

            // 删除指定范围元素，含头不含尾，成功返回null
            fruits.removeRange(start,end);

            // 删除指定条件的元素，成功返回null
            fruits.removeWhere((item) => item.length >6)；

            // 删除所有的元素
            fruits.clear();

            // sort()对元素进行排序，传入一个函数作为参数，return <0表示由小到大， >0表示由大到小
            fruits.sort((a, b) => a.compareTo(b));

###### map
> 类似 JavaScript map

            // Map的声明
            var hawaiianBeaches = {
                'oahu' : ['waikiki', 'kailua', 'waimanalo'],
                'big island' : ['wailea bay', 'pololu beach'],
                'kauai' : ['hanalei', 'poipu']
            };
            var searchTerms = new Map();

            // 指定键值对的参数类型
            var nobleGases = new Map<int, String>();

            // Map的赋值，中括号中是Key，这里可不是数组
            nobleGase[54] = 'dart';

            //Map中的键值对是唯一的
            //同Set不同，第二次输入的Key如果存在，Value会覆盖之前的数据
            nobleGases[54] = 'xenon';
            assert(nobleGases[54] == 'xenon');

            // 检索Map是否含有某Key
            assert(nobleGases.containsKey(54));

            //删除某个键值对
            nobleGases.remove(54);
            assert(!nobleGases.containsKey(54));

注：如果定义了一个 map 常量，那么value 也必须是常量

###### symbol
>symbol字面量是编译时常量，在标识符前面加#。如果是动态确定，则使用Symbol构造函数，通过new来实例化
---

### 函数
>所有的函数都会有返回值。如果没有指定函数返回值，则默认的返回值是null。没有返回值的函数，系统会在最后添加隐式的return 语句。

###### 函数参数

函数可以有两种类型的参数：
* 必须的——必须的参数放在参数列表的前面。
* 可选的——可选的参数跟在必须的参数后面。

注：可选参数必须放在最后
>通过【】来表示可选参数

            String say(String from, String msg, [String device]) {
              var result = '$from says $msg';
              if (device != null) {
                result = '$result with a $device';
              }
              return result;
            }

>还可以设置默认参数值

            String say(String from, String msg,
                [String device = 'carrier pigeon', String mood]) {
              var result = '$from says $msg';
              if (device != null) {
                result = '$result with a $device';
              }
              if (mood != null) {
                result = '$result (in a $mood mood)';
              }
              return result;
            }

>函数还可以作为另一个函数的参数

            printElement(element) {
              print(element);
            }

            var list = [1, 2, 3];

            // Pass printElement as a parameter.
            list.forEach(printElement);

>函数可以匿名，但是不像 JavaScript， 匿名函数不用加上 function 关键字

            var list = ['apples', 'oranges', 'grapes', 'bananas', 'plums'];
            list.forEach((i) {
              print(list.indexOf(i).toString() + ': ' + i);
            });

###### 闭包

            Function makeAdder(num addBy) {
              return (num i) => addBy + i;
            }

            main() {
              // Create a function that adds 2.
              var add2 = makeAdder(2);

              // Create a function that adds 4.
              var add4 = makeAdder(4);

              assert(add2(3) == 5);
              assert(add4(3) == 7);
            }

---

### 运算符
除了常见的，还有如下运算符：
* is 运算符，a is b，用于判断 a 对象是否是 b 类的实例，返回 bool 值
* is！意义与上面相反
* as 运算符；用于检查类型

            (emp as Person).firstName = 'Bob';

如果 emp 为空或者不是 Person 的实例，会抛出异常
* ??= 运算符

            b ??= value; // 如果 b 为空，把 value 赋值给 b;
                         // 否则，b 不变

* ?? 运算符

            String toString() => msg ?? super.toString();
            //如果 msg 不为空，返回 msg；否则返回后面的

* .. 运算符，把对同一对象的不同操作串联起来

            final addressBook = (new AddressBookBuilder()
                  ..name = 'jenny'
                  ..email = 'jenny@example.com'
                  ..phone = (new PhoneNumberBuilder()
                        ..number = '415-555-0100'
                        ..label = 'home')
                      .build())
                .build();

### 流程控制语句
* if...else
* for
* while do-while
* break continue
* switch...case  如果 case 后面有表达式但是没有 break，会抛出异常
* assert（仅在checked模式有效），如果条件为假，抛出异常

---

### 异常
##### throw
* 抛出固定类型的异常：

            throw new FormatException('Expected at least 1 section');
* 抛出任意类型的异常：

            throw 'out of llamas！'
* 因为抛出异常属于表达式，可以将throw语句放在=>语句中，或者其它可以出现表达式的地方：

            distanceTo(Point other) =>
                throw new UnimplementedError();
##### catch
* 可以通过 on语句来指定需要捕获的异常类型，使用catch来处理异常

            try {
              breedMoreLlamas();
            } on OutOfLlamasException {
              // A specific exception
              buyMoreLlamas();
            } on Exception catch (e) {
              // Anything else that is an exception
              print('Unknown exception: $e');
            } catch (e, s) {
              print('Exception details:\n $e');
              print('Stack trace:\n $s');
            }

可以向catch()传递1个或2个参数。第一个参数表示：捕获的异常的具体信息，第二个参数表示：异常的堆栈跟踪(stack trace)

##### rethrow
>rethrow语句用来处理一个异常，同时希望这个异常能够被其它调用的部分使用

##### finally
>Dart 的finally用来执行那些无论异常是否发生都执行的操作。

---

### 类
>使用new语句来构造一个类。构造函数的名字可能是ClassName，也可以是ClassName.identifier

            var jsonData = JSON.decode('{"x":1, "y":2}');

            // Create a Point using Point().
            var p1 = new Point(2, 2);

            // Create a Point using Point.fromJson().
            var p2 = new Point.fromJson(jsonData);

* 使用.来调用实例的变量或者方法。
* 使用 ?. 来避免左边操作数为null引发异常。
* 使用const替代new来创建编译时的常量构造函数。
* 两个使用const构建的同一个构造函数，实例相等。
* 获取对象的运行时类型使用：o.runtimeType

>所有实例变量会生成一个隐式的getter方法，不是final或const的实例变量也会生成一个隐式的setter方法

##### 构造函数

            class Point {
              num x;
              num y;

              // 推荐方式
              Point(this.x, this.y);
            }

>构造函数不能被继承

子类不会继承父类的构造函数。如果不显式提供子类的构造函数，系统就提供默认的构造函数。

>命名构造函数

            class Point {
              num x;
              num y;

              Point(this.x, this.y);

              // 命名构造函数
              Point.fromJson(Map json) {
                x = json['x'];
                y = json['y'];
              }
            }

使用命名构造函数可以实现一个类多个构造函数。构造函数不能被继承，父类中的命名构造函数不能被子类继承。如果想要子类也拥有一个父类一样名字的构造函数，必须在子类实现这个构造函数

>如果父类不显式提供无参的非命名构造函数，在子类中必须手动调用父类的一个构造函数。在子类构造函数名后，大括号{前，使用super.调用父类的构造函数，中间使用:分割

            class Person {
              String firstName;

              Person.fromJson(Map data) {
                print('in Person');
              }
            }

            class Employee extends Person {
              // 父类没有无参数的非命名构造函数，必须手动调用一个构造函数 super.fromJson(data)
              Employee.fromJson(Map data) : super.fromJson(data) {
                print('in Employee');
              }
            }

>当在构造函数初始化列表中使用super()时，要把它放在最后。

            View(Style style, List children)
                : _children = children,
                  super(style) {}

>除了调用父类的构造函数，也可以通过初始化列表 在子类的构造函数体前（大括号前）来初始化实例的变量值，使用逗号,分隔

            class Point {
              num x;
              num y;

              Point(this.x, this.y);

              // 在构造函数体前 初始化列表 设置实例变量
              Point.fromJson(Map jsonMap)
                  : x = jsonMap['x'],
                    y = jsonMap['y'] {
                print('In Point.fromJson(): ($x, $y)');
              }
            }

##### 工厂构造函数
>当实例化了一个构造函数后，不想每次都创建该类的一个新的实例的时候使用factory关键字，定义工厂构造函数，从缓存中返回一个实例，或返回一个子类型的实例

            class Logger {
              final String name;
              bool mute = false;
              static final Map<String, Logger> _cache = <String, Logger>{}; // 缓存保存对象
              factory Logger(String name) {
                if (_cache.containsKey(name)) {
                  return _cache[name];
                } else {
                  final logger = new Logger._internal(name);
                  _cache[name] = logger;
                  return logger;
                }
              }
              Logger._internal(this.name);// 命名构造函数
              void log(String msg) {
                if (!mute) {
                  print(msg);
                }
              }
            }

            main() {
              var p1 = new Logger("1");
              p1.log("2");

              var p2 = new Logger('22');
              p2.log('3');
              var p3 = new Logger('1');// 相同对象直接访问缓存
            }

##### 方法
###### Getters and setters
>get()和set()方法是Dart 语言提供的专门用来读取和写入对象的属性的方法。每一个类的实例变量都有一个隐式的getter和可能的setter（如果字段为final或const，只有getter）

##### 抽象类
>使用abstract关键字定义一个抽象类，抽象类不能实例化。抽象类通常用来定义接口。

##### 隐式接口
>每一个类都隐式的定义一个接口，这个接口包含了这个类的所有实例成员和它实现的所有接口

>一个类可以实现一个或多个（用,隔开）接口，通过implements关键字。

            class Person {
              final _name;
              Person(this._name);
              String greet(who) => 'hello,$who,i am $_name';
            }

            class Imposter implements Person {
              final _name = '';
              String greet(who) => 'hi $who.do you know who i am.';
            }

            greetBob(Person p) => p.greet('bob');
            main(List<String> args) {
              print(greetBob(new Person('lili')));
              print(greetBob(new Imposter()));
            }

##### 继承
>使用extends来创造子类，使用super来指向父类

##### 枚举类型
>枚举类型是一种特殊的类，通常用来表示一组固定数字的常量值。

>每个枚举类型都有一个index的getter，返回以0开始的位置索引，每次加1。

>在switch语句中使用枚举，必须在case语句中判断所有的枚举，否则会获得警告。

枚举类型有以下限制：
* 不能继承，mixin，或实现一个枚举。
* 不能显式的实例化一个枚举。

---

### 泛型

>使用<...> 的方式来定义泛型

>虽然Dart 语言中类型是可选的，但是明确的指明使用的是泛型，会让代码更好理解

            abstract class Cache<T> {
               T getByKey(String key);
               setByKey(String key, T value);
             }

##### 用于集合类型
>泛型用于List 和 Map 类型参数化

            var names = <String>['Seth', 'Kathy', 'Lars'];
            var pages = <String, String>{
              'index.html': 'Homepage',
              'robots.txt': 'Hints for web robots',
              'humans.txt': 'We are people, not machines'
            };

##### 泛型集合及它们所包含的类型
>dart的泛型类型是具体的，在运行时包含它们的类型信息。

---

### 库和可见性
>使用import 和 library 指令可以方便的创建一个模块或分享代码。一个Dart 库不仅能够提供相应的API，还可以包含一些以_开头的私有变量仅在库内部可见

>如果导入的库拥有相互冲突的名字，使用as为其中一个或几个指定不一样的前缀。

            import 'package:lib1/lib1.dart';
            import 'package:lib2/lib2.dart' as lib2;
            // ...
            Element element1 = new Element();           // Uses Element from lib1.
            lib2.Element element2 = new lib2.Element(); // Uses Element from lib2.

>如果只需要使用库的一部分内容，使用show或hide有选择的导入。

            // 仅导入foo.
            import 'package:lib1/lib1.dart' show foo;

            // 除了foo都导入
            import 'package:lib2/lib2.dart' hide foo;

>要延迟加载一个库，首先必须使用deferred as导入它。

            import 'package:deferred/hello.dart' deferred as hello;

            greet() async {
              // 使用await关键字暂停执行，直到库加载
              await hello.loadLibrary();
              hello.printGreeting();
            }

可以在代码中多次调用loadLibrary()方法。但是实际上它只会被执行一次。

使用延迟加载的注意事项：

* 延迟加载的内容只有在加载后才存在。
* Dart 隐式的将deferred as改为了deferred as namespace。loadLibrary()返回值是Future

---
### 异步支持
使用async函数和await表达式实现异步操作。

当需要使用一个从Future返回的值时，有两个选择：

* 使用async和await。
* 使用Future API。

当需要从一个Stream获取值时，有两个选择：

* 使用async和异步的循环(await for)。
* 使用Stream API。
代码使用了async和await就是异步的，虽然看起来像同步代码。

            checkVersion() async {          //注意这里 async 在小括号后面，和 JavaScript 不一样
              var version = await lookUpVersion();
              if (version == expectedVersion) {
                // Do something.
              } else {
                // Do something else.
              }
            }

>给函数添加async关键字将使函数返回一个Future类型。

            // 修改前是同步的
            String lookUpVersionSync() => '1.0.0';

            // 修改后 是异步的 函数体不需要使用Future API
            // dart会在必要的时候创建Future对象
            Future<String> lookUpVersion() async => '1.0.0';

>在Stream中使用异步循环

             // expression的值必须是Stram类型
            await for (variable declaration in expression) {
              // Executes each time the stream emits a value.
            }

异步循环的执行流程如下：

* 等待 stream 发出数据。
* 执行循环体，并将变量的值设置为发出的数据。
* 重复1.，2.直到stream 对象被关闭

注：这个过程类似于 JavaScript 的 Rxjs

---
### 可调用类
>Dart 语言中为了能够让类像函数一样能够被调用，可以实现call()方法。

            class WannabeFunction {
              call(String a, String b, String c) => '$a $b $c!';
            }

            main() {
              var wf = new WannabeFunction();
              var out = wf("Hi","there,","gang");
              print('$out'); // Hi there, gang!
              print(wf.runtimeType); // WannabeFunction
              print(out.runtimeType); // String
              print(wf is Function); // true
            }