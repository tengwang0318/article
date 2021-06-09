## Thoughts about class in python

### 1. Private attribute

在python用两个下划线可以用来表示private类型，但根据python的设计理念为"We are all consenting adluts"，你依然可以在无需调用get 或 set方法对private类型的变量进行修改。

```python
class Solution:
    __nb = "qwerdf"

    def get(self):
        return self.__nb


test = Solution()

# __nb is private attribute. You will get an error about you can't find the attribute about __nb
try:
    print(test.__nb)
except AttributeError as e:
    print(e)
# but you can use get method to attain __nb
# you can use public method to attain __nb
print(test.get())

# Python theory is "We are all consenting adults".
# You can use some tricks to realize to get and set private attribute without public method
# You can use classname._name__privateAttribute to get and set
print(test._Solution__nb)
test._Solution__nb = "🐂"
print(test._Solution__nb)
```

结果如下：

```
'Solution' object has no attribute '__nb'
qwerdf
qwerdf
🐂
```



### 2. Use super() or `__init__()` in muti-inheritance

多重继承中使用super()的两个原因：

#### 1. 使用`__init__()`时，初始化的顺序是子类的实际调用顺序，而不是“导入”字累的顺序

```python
class MyBaseClass(object):
    def __init__(self, value):
        self.value = value


class TimesTwo(object):
    def __init__(self):
        self.value *= 2


class PlusFive(object):
    def __init__(self):
        self.value += 5


class OneWay(MyBaseClass, TimesTwo, PlusFive):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        TimesTwo.__init__(self)
        PlusFive.__init__(self)


class AnotherWay(MyBaseClass, PlusFive, TimesTwo):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        TimesTwo.__init__(self)
        PlusFive.__init__(self)


test1 = OneWay(5)
print("First order is 5 * 2 + 5 =", test1.value)
test2 = AnotherWay(5)
print("Another order is also 5 * 2  + 5 =", test2.value)


```

输出结果

```
First order is 5 * 2 + 5 = 15
Another order is also 5 * 2  + 5 = 15
```

以OneWay为例，他首先初始化MyBaseClass类，给OneWay的 self.value 设为5，然后执行TimesTwo的构造函数，即$self.value = self.value * 2 = 5 * 2$

AnotherWay“导入”超类的顺序为MyBaseClass, PlusFive , TimeTwo与OneWay“导入”超类的顺序不同，但AnotherWay构造函数执行的顺序与OneWay的构造函数执行顺序一致。

```python
class MyBaseClass(object):
    def __init__(self, value):
        self.value = value


class TimesTwo(MyBaseClass):
    def __init__(self, value):
        super().__init__(value)
        self.value *= 2


class PlusFive(MyBaseClass):
    def __init__(self, value):
        super().__init__(value)
        self.value += 5


class OneWay(PlusFive, TimesTwo):
    def __init__(self, value):
        super(OneWay, self).__init__(value)


class AnotherWay(TimesTwo, PlusFive):
    def __init__(self, value):
        super(AnotherWay, self).__init__(value)



test1 = OneWay(5)
print("First order is 5 * 2 + 5 = ", test1.value)
test2 = AnotherWay(5)
print("Another order is also (5 + 5) * 2 = ", test2.value)
```

输出结果：

```
First order is 5 * 2 + 5 =  15
Another order is also (5 + 5) * 2 =  20
```

可以看出，使用super()在多重继承中初始化顺序就是“导入”包的顺序，与`__init__()`不同。

可以调用mro函数，对其超类的倒入顺序进行查看，mro为method resolution order，即为方法执行顺序

运行代码

```python
print(OneWay.mro())
print(AnotherWay.mro())
```

```
[<class '__main__.OneWay'>, <class '__main__.PlusFive'>, <class '__main__.TimesTwo'>, <class '__main__.MyBaseClass'>, <class 'object'>]
[<class '__main__.AnotherWay'>, <class '__main__.TimesTwo'>, <class '__main__.PlusFive'>, <class '__main__.MyBaseClass'>, <class 'object'>]
```

以OneWay为例，说明mro的意义：

从结果上来看，调用OneWay导入的反方向运行的，可以这样来理解，要想调用OneWay，就必须要先执行PlusFive，要想执行PlusFive就要执行MyBaseClass才能获取相应的值，要想执行MyBaseClass就要先执行object，就不难理解OneWay导入超类时，是反向的。（个人理解，勿喷）

#### 2.多重继承问题。

例子为一个子类继承两个单独超类，两个超类都继承于同一个公共基类

```python
class MyBaseClass(object):
    def __init__(self, value):
        self.value = value


class TimesFive(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value *= 5


class PlusTwo(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value += 2


class ThisWay(TimesFive, PlusTwo):
    def __init__(self, value):
        TimesFive.__init__(self, value)
        PlusTwo.__init__(self, value)


test = ThisWay(5)
print("Should 5 * 5 + 2 = ", test.value)
```

输出结果：

```
Should 5 * 5 + 2 =  7
```

原因在于ThisWay的构造函数先调用TimesFive的构造函数,此时self.value = 5 * 5 = 25

ThisWay的构造函数会继续调用PlusTwo的构造函数,PlusTwo调用自身的构造函数时，会调用MyBaseClass的构造函数，就会将self.value重置为5，而不是25，这就是结果与预期相差的原因。

当然这种可以让调用TimeFive & PlusTwo构造函数调用时return一个值进行解决

```python
class MyBaseClass(object):
    def __init__(self, value):
        self.value = value


class TimesFive(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value *= 5
        return self.value


class PlusTwo(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value += 2
        return self.value


class ThisWay(TimesFive, PlusTwo):
    def __init__(self, value):
        value = TimesFive.__init__(self, value)
        PlusTwo.__init__(self, value)


test = ThisWay(5)
print("Should 5 * 5 + 2 = ", test.value)

```

输出结果为：

```
Should 5 * 5 + 2 =  27
```

使用super()来进行解决:



```python
class MyBaseClass(object):
    def __init__(self, value):
        self.value = value


class TimesFive(MyBaseClass):
    def __init__(self, value):
        super(TimesFive, self).__init__(value)
        self.value *= 5


class PlusTwo(MyBaseClass):
    def __init__(self, value):
        super(PlusTwo, self).__init__(value)
        self.value += 2


class ThisWay(PlusTwo, TimesFive):
    def __init__(self, value):
        super(ThisWay, self).__init__(value)


test = ThisWay(5)
print("Should 5 * 5 + 2 = ", test.value)

```

输出结果为：

```
Should 5 * 5 + 2 =  27
```

调用mro() ,查看超类导入顺序，

```
print(ThisWay.mro())
```

输出结果(从右往左为调用顺序？)：先执行TimesFive的构造函数，才执行PlusTwo的构造函数。

```
[<class '__main__.ThisWay'>, <class '__main__.PlusTwo'>, <class '__main__.TimesFive'>, <class '__main__.MyBaseClass'>, <class 'object'>]

```

这样就没有使用`__init__()`出现钻石继承问题



此时，您可能会有个疑问，上述使用super()的情况都是只有一个属性值，如果两个超类有不互相同的属性，那么该如何操作呢？



就以课本上例子为例进行改造，

```python
class People:
    name = ""
    age = 0
    __weight = 0

    def __init__(self, n, a, w):
        self.name = n
        self.age = a
        self.__weight = w


class Student(People):
    grade = ""

    def __init__(self, n, a, w, g):
        super(Student, self).__init__(n, a, w)
        self.grade = g


class Speaker(People):
    topic = ""

    def __init__(self, n, a, w, t):
        super(Speaker, self).__init__(n, t, a, w)
        self.topic = t

class Sample(Speaker, Student):
    def __init__(self):
    #不知道如何传参数，因为Speaker与Student的构造函数的参数不同
        super(Sample, self).__init__()

```

 此时就不知道如何给Sample的构造函数传参数，因为Speaker与Student的构造函数的参数不同。

参考[StackOverFlow](https://stackoverflow.com/questions/26927571/multiple-inheritance-in-python3-with-different-signatures)

```python
class People:
    name = ""
    age = 0
    __weight = 0

    def __init__(self, n="", a=0, w="", *args, **kwargs):
        self.name = n
        self.age = a
        self.__weight = w


class Student(People):
    grade = ""

    def __init__(self, n="", a=0, w="", g=0, *args, **kwargs):
        super(Student, self).__init__(n=n, a=a, w=w, *args, **kwargs)
        self.grade = g


class Speaker(People):
    topic = ""

    def __init__(self, n="", a=0, w=0, t="", *args, **kwargs):
        super(Speaker, self).__init__(n=n, t=t, a=a, w=w, *args, **kwargs)
        self.topic = t


class Sample(Student, Speaker):
    def __init__(self, n, a, w, g, t):
        super(Sample, self).__init__(n=n, a=a, w=w, g=g, t=t)


test = Sample(n="WangTeng", a=18, w=130, g=1000, t="Dont' be a 奋斗逼")

print(test.name, test.age, 1, test.grade, test.topic)
print(Sample.mro())

```

输出结果Bingo完美解决：

```
WangTeng 18 1 1000 Dont' be a 奋斗逼
[<class '__main__.Sample'>, <class '__main__.Student'>, <class '__main__.Speaker'>, <class '__main__.People'>, <class 'object'>]
```



