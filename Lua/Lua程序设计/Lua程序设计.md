### 第1章 开始
#### 1.1 程序块
- 几条连续的Lua语句之间并不需要分隔符，但如果愿意，也可以使用分号来分隔语句。例如，当两条或多条语句并列出现在同一行时使用分号分隔它们。在Lua的语法中，代码中的换行不起任何作用。

#### 1.2 词法规范
- Lua中的标识符可以是由任意字母、数字和下划线构成的字符串，但不能以数字开头
- 应该避免使用以一个下划线开头并跟着一个或者多个大写字母（例如“**_VERSION**”）的标识符，Lua将这类标识符保留用作特殊用途。通常保留标识符“**-**”（一个下划线）作为“哑变量”使用
- 可以在任何地方以两个连字符（--）开始一个“行注释”，该注释一直延伸到一行的结尾。Lua也提供了“块注释”，以“--[[”开始，直至"--]]"。当注释一段代码时，一个常见的技巧是将这些代码放入"--[["和“--]]”中间。
    ```
    --[[
    print(10)
    --]]
    ```
    当重新启用这段代码时，只需在第一行行首添加一个连字符即可：
    ```
    ---[[
    print(10)
    --]]
    ```

#### 1.3 全局变量
- 在Lua中，访问一个未初始化的变量不会引发错误，访问结果是一个特殊的值nil。如果要删除某个全局变量的话，只需将其赋值为nil。

#### 1.4 解释器程序
- **如果代码文件的第一行以一个井号（#）开头，那么在加载该文件时，解释器将忽略这一行**。这项特征主要是为了方便在UNIX系统中将Lua作为一种脚本解释器来使用。如果用下面这行开始脚本代码的编写：
    ```
    #!/usr/local/bin/lua
    ```
    或
    ```
    #!/usr/bin/env lua
    ```
    那么便可以直接调用脚本文件，而无须显式地调用Lua解释器了。

- 在解释器执行其参数前，会先查找一个名为LUA_INIT的环境变量，如果找到了这个变量，并且其内容为“@文件名”，那么解释器会先执行这个文件。如果LUA_INIT没有以“@”开头，那么解释器就假设变量内容为Lua代码，并运行此代码。由于LUA_INIT可以很灵活地配置解释器，并且完全可以控制如何配置它。例如，可以预先加载一个程序包、修改命令提示符和路径、定义函数、对函数进行改名或删除等。

- 在脚本代码中，可以通过全局变量arg来检索脚本的启动参数。例如：
    ```
    % lua 脚本 a b c
    ```
    解释器在运行脚本前，会用所有的命令行参数创建一个名为“arg”的table。脚本名称位于索引0上，它的第一个参数（示例中的“a”）位于索引1，以此类推。而在“脚本”之前的所有选项参数则位于负数索引上。举例如下：
    ```
    % lua -e "sin=math.sin" script a b
    ```
    解释器将所有参数组织排列为：
    ```lua
      arg[-3] = "lua"
      arg[-2] = "-e"
      arg[-1] = "sin=math.sin"
      arg[0]  = "script"
      arg[1]  = "a"
      arg[2]  = "b"
    ```

### 第2章 类型与值
在lua中有8中基础类型：nil、boolean、number、string、userdata、function、thread、table。函数type可根据一个值返回其类型名称。
```lua
print(type("hello world"))  --> string
print(type(10.4*3))         --> number
print(type(print))          --> function
print(type(type))           --> function
print(type(true))           --> boolean
print(type(nil))            --> nil
print(type(type(X)))        --> string
```
**将一个变量用于不同类型，通常会导致混乱的代码，但有时明智地使用这种特性会带来便利。例如，在异常情况下，可以返回一个nil**

#### 2.1 nil
将nil赋予一个变量等同于删除它

#### 2.2 boolean
lua将值false和nil视为假，而将除此之外的其他值视为“真”

#### 2.3 number
有人会担心即使对浮点数进行一个简单的递增运算都有可能导致错误的结果。而事实上，只要使用双精度来表示一个整数，就不会出现“四舍五入”的错误

#### 2.4 string
- 可以使用一对匹配的双括号来界定一个字母字符串，就像写“块注释”那样。以这种形式书写的字符串可以延伸多行，Lua不会解释其中的转义序列。这种书写法对于书写那种含有程序代码的字符串尤为有用。
  ```lua
    page = [[
    <html>
    <head>
    <title>An HTML Page</tilte>
    </head>
    <body>
    <a href="http://www.lua.org">Lua</a>
    </body>
    </html>
    ]]
  ```
- Lua提供了运行时的数字与字符串的自动转换。在一个字符串上应用算术操作时，Lua会尝试将这个字符串转换成一个数字。
  ```lua
    print("10"+1)     ---> 11
  ```

- 在Lua中,".."是字符串连接操作符。当直接在一个数字后面输入它的时候，必须要用一个空格来分隔它们。不然，Lua会将第一个点理解为一个小数点。
- 使用tonumber函数可以将字符串转换成数字
- 若要将一个数字转换成字符串，可以调用函数tostring，或者将该数字与一个空字符串相连接
- 使用“#”获得字符串长度
  ```lua
    print(#"good\0bye")   --> 8
  ```

#### 2.5 table
- table永远是“匿名的”，一个持有table的变量与table自身之间没有固定的关联性
  ```lua
    a = {}
    a["x"] = 10
    b = a           --> b与a引用了同一个table
    print(b["x"])   --> 10
    b["x"] = 20
    print(a["x"])   --> 20
    a = nil         -- 现在只有b还引用table
    b = nil         -- 再也没有对table的引用了
  ```
- **初学者常会将a.x和a[x]搞错。前者表示a["x"],表示以字符串“x”来索引table。而后者是以变量x的值来索引table**
  ```lua
    i = 10; j = "10";
    a[i] = "one value"
    a[j] = "another value"
    print(a[i])              --> one value
    print(a[j])              --> another value
    print(a[tonumber(j)])    --> one vlaue
  ```
- Lua数组通常以1作为索引的起始值。并且还有不少机制依赖于这个惯例。
- 在lua5.1中，长度操作符“#”用于返回一个数字或线性表的最后一个索引值。
- 请记住对于所有未初始化的元素的索引结果都是nil。Lua将nil作为界定数组结尾的标示。当一个数组有“空隙”时，即中间含有nil时，长度操作符会认为这些nil元素就是结尾标记。因此应该避免对那些含有“空隙”的数组使用长度操作符。

#### 2.6 function
#### 2.7 userdata和thread

### 第3章 表达式
#### 3.1 算术操作符
#### 3.2 关系操作符
对于table、userdata和函数，lua是作引用比较的。也就是说，只有当它们引用同一个对象时，才认为它们相等
#### 3.3 逻辑操作符
- **对于and操作符来说，如果它的第一个操作数为假，就返回第一个操作数；不然返回第二个操作数。对于操作符or来说，如果它的第一个操作数为真，就返回第一个操作数；不然返回第二个操作数。**
  ```lua
    print(4 and 5)         --> 5
    print(nil and 13)      --> nil
    print(false and 13)    --> false
    print(4 or 5)          --> 4
    print(false or 5)      --> 5
  ```
- 有一种常用的Lua习惯写法“x=x or v”,它等价于：
  ```lua
    if not x then x = v end
  ```
    它可用在没有设置X的时候（即对x的求值结果为假时），将其设为一个默认值V。
- 还有一种习惯写法是“(a and b) or c”,这类似于C语言中的表达式“a ? b : c”,但是前提是b不为假。例如，为了选出数字x和y中的较大者，可以使用以下语句：
  ```lua
    max = (x > y) and x or y
  ```
    若x>y, 则and的第一个操作数为真。那么and运算的结果就是其第二个操作数x，而x是一个永远为真的表达式。然后or运算的结果就是其第一个操作数x。当x>y为假的时候，and表达式为假，因此or的结果是其第二个操作数y。

#### 3.4 字符串连接
使用操作符".."（两个点）可以连接两个字符串。如果其任意一个操作数是数字的话，lua会将这个数字转换成一个字符串。

**Lua中的字符串是不可变得值。连接操作符只会创建一个新字符串，而不会对其原操作数进行任何修改**

```lua
a = "hello"
print(a.." world")   --> hello world
print(a)             --> hello
```

#### 3.5 优先级
#### 3.6 table构造式
- 构造式是用于创建和初始化table的表达式；最简单的构造式就是一个空构造式{}，用于创建一个空table。

```lua
days = {"Sunday", "Monday"}   --> 等价于days={};days[1]="Sunday";days[2]="Monday" 
a = {x=10, y=20}              --> 等价于 a={}; a.x=10; a.y=20
```
- 对于某些情况如果真的需要以0作为一个数组的起始索引的话，通过这种语法也可以轻松做到：
  ```lua
    days = {[0]="Sunday", "Monday", "Tuesday", "Wednesday"}
  ```
    **不推荐在lua中以0作为数组的起始索引。大多数内键函数都假设数组起始于索引1**

- 最后，在一个构造式中还可以用分号代替逗号。通常会将分号用于分隔构造式中不同的成分，例如将列表部分与记录部分明显地区分开：
  ```lua
    {x=10, y=45; "one", "two", "three"}
  ```

### 第4章 语句

#### 4.1 赋值
Lua允许“多重赋值”，如：
```lua
a,b = 10, 2*x  --> a为10， b为2*x
```
Lua先对等号右边的所有元素求值，然后才执行赋值。
```
x,y = y,x    ---> 交换x与y
```
Lua总是会将等号右边值得个数调整到与左边变量的个数相一致。规则是：若值得个数少于变量的个数，那么多余的变量会被赋值为nil；若值得个数更多的话，那么多余的值会被丢弃掉：
```lua
a,b,c = 0, 1
print(a, b, c)      ---> 0 1 nil
a, b = a+1, b+1, b+2 -- b+2被忽略
print(a, b)         ---> 1 2
a, b, c = 0
print(a, b, c)      ---> 0, nil, nil
```
多重赋值比较常见的应用是用于收集函数的多个返回值。

#### 4.2 局部变量与块
- 通过local语句来创建局部变量
- 如果要严格地控制某些局部变量的作用域，可以使用do块
  ```lua
    do
      local a2 = 2*a
      local d = b^2
      x1 = (-b + d)/a2
      x2 = (-b - d)/a2
    end     -- a2和d的作用域到此结束
    print(x1, x2)
  ```
- **尽可能地使用局部变量是一种良好的编程风格**局部变量可以避免将一些无用的名称引入全局环境，避免搞乱全局环境。此外，**访问局部变量比访问全局变量更快。局部变量会随着其作用域的结束而消失，这样便使垃圾收集器可以释放其值**

- 在Lua中，有一种习惯写法是：
  ```lua
        local foo = foo
  ```
    这句代码创建了一个局部变量foo， 并用全局变量foo的值初始化它。这种方式可以加速在当前作用域中对foo的访问。

#### 4.3 控制结构
**Lua将所有不是false和nil的值视为“真”**

##### 4.3.1 if then else
##### 4.3.2 while
##### 4.3.3 repeat-until
##### 4.3.4 数字型for
for语句有两种形式：数字型for和泛型for。
数字型for的语法如下：
```lua
for var=exp1, exp2, exp3 do
    <执行体>
end
```
var从exp1变化到exp2，每次变化都以exp3作为步长递增var，并执行一次“执行体”。第三个表达式exp3是可选的，若不指定的话，默认为1.
```lua
for i=1, f(x) do print(i) end
```
for的3个表达式是在循环开始前一次性求值。例子，上列中的f(x)只会执行一次。控制变量会被自动地声明为for语句的局部变量，仅在循环体内可见。
##### 4.3.5 泛型for
泛型for循环通过一个迭代器函数来遍历所有值。
```lua
for i,v in ipairs(a) do print(v) end
```
- 标准库提供了几种迭代器
  - io.lines      （迭代文件中每行）
  - pairs         （迭代table元素）
  - ipairs        （迭代数组元素）
  - string.gmatch （迭代字符串中单词）


#### 4.4 break与return

### 第5章 函数
- 函数参数要放到一对圆括号中。有一种特殊情况例外：一个函数若只有一个参数，并且此参数是一个字母字符串或table构造式，那么圆括号便是可有可无。
  ```lua
    print "hello world"
    dofile 'a.lua'
    f {x=10, y=20}
    type {}
  ```
- Lua为面向对象式的调用提供了一种特殊的语法-冒号操作符。**表达式o.foo(o, x)的另一种写法是o:foo(x)**,冒号操作符使调用o.foo时将o隐含地作为函数的第一个参数。
- **调用函数时提供的实参数量可以与形参数量不同**。Lua会自动调整实参的数量，以匹配参数表的要求。**若实参多余形参，则舍弃多余的实参；若实参不足，则多余的形参初始化为nil**

#### 5.1 多重返回值
- 当一个函数调用作为另一个函数调用的**最后一个实参时**，第一个函数的所有返回值都将作为实参传入第二个函数。
  ```lua
    print(foo0())       -->
    print(foo1())       --> a
    print(foo2())       --> a b
    print(foo2(), 1)    --> a 1
    print(foo2().."x")  --> ax
  ```
    **当foo2出现在第一个表达式中时，Lua会将其返回数量调整为1**.以此上面最后一行中，只有“a”参与了字符串连接操作。
- **可以将一个函数调用放入一对圆括号中，从而迫使它只返回一个结果**：
  ```lua
    print((foo0()))   --> nil
    print((foo1()))   --> a
    print((foo2()))   --> a
  ```
- unpack 接受一个数组作为参数，并从下标1开始返回该数组的所有元素：
  ```lua
    print(unpack(10, 20, 30))   --> 10 20 30
    a, b = unpack {10， 20， 30} -- a=10, b=20, 30被丢弃
  ```
    如果想调用任意函数f，而所有的参数都在数组a中，那么可以这么写：
  ```lua
    f(unpack(a))
  ```

#### 5.2 变长参数
```lua
function add(...)
    local s = 0
    for i, v in ipairs{...} do
        s = s + v
    end
    return s
end
```
参数表中的3个点(...)表示该函数可以接受不同数量的实参。表达式“...”的行为类似于一个具有多重返回值得函数，它返回的是当前函数的所有变长参数。例如：
```lua
 local a, b = ...
```

- 如果变长参数包含nil，则需要使用select来访问变长参数。
  ```lua
    for i=1, select('#', ...) do
        local arg = select(i, ...)  -- 得到第i个参数
        <循环体>
    end
  ```
    **select("#", ...)会返回所有变长参数的总数，其中包括nil**

#### 5.3 具名实参
```lua
-- 无效的演示代码
rename(old="temp.lua", new="temp1.lua")
```
Lua中不直接支持这种语法，但可以通过一种细微的改变来获得相同的效果。主要是将所有实参组织到一个table中，并将这个table作为唯一的实参传给函数。
```lua
rename {old="temp.lua", new="temp1.lua"}
```
另一方面，将rename改为只接受一个参数，并从这个参数中获取实际的参数：
```lua
function rename(arg)
    return os.rename(arg.old, arg.new)
end
```
若一个函数拥有大量的参数，而其中大部分参数是可选的话，这种参数传递风格会特别有用。例如在一个GUI库中，一个用于创建新窗口的函数可能会具有许多的参数，而其中大部分都是可选的，那么最好使用具名实参：
```lua
w = Window{x=0, y=0, width=300, height=200,title="Lua",
    background="blue", border=true
}
```
Window函数可以根据要求检查一些必填参数，或者为某些参数添加默认值。加上“_Window”才是真正用于创建新窗口的函数，它要求所有参数以正确的次序传入，那么Windows函数可以这么写：
```lua
function Window(options)
    -- 检查必要的参数
    if type(options.title) ~= "string" then
        error("no title")
    elseif type(options.width) ~= "number" then
        error("no width")
    elseif type(options.height) ~= "number" then
        error("no height")
    -- 其他参数都是可选的
    _Windows(options.titles,
             options.x or 0,                  -- 默认值
             options.y or 0,                  -- 默认值
             options.width, options.height,
             options.background or "white",   -- 默认值
             options.border                   -- 默认值false(nil)
    )
end
```

### 第6章 深入函数
- 在Lua中函数与其他传统类型的值具有相同的权利。函数可以存储到变量中或table中，可以作为实参传递给其他函数，还可以作为其他函数的返回值。
- Lua中最常见的是函数编写方式，诸如：
  ```lua
    function foo(x) return 2*x end
  ```
    只是一种所谓的“语法糖”而已。这只是以下代码的一种简化书写形式：
  ```lua
    foo = function(x) return 2*x end
  ```

#### 6.1 closure(闭合函数)
- 若将一个函数写在另一个函数之内，那么这个位于内部的函数便可以访问外部函数中的局部变量，这项特征称之为“词法域”。
- **非局部的变量**，在下例子中，传递给sort的匿名函数可以访问参数grades，而grades是外部函数sortbygrade的局部变量。在这个匿名函数内部，grades既不是全局变量也不是局部变量，将其称为一个“非局部的变量”
  ```lua
    function sortbygrade(names, grades)
        table.sort(names, function(n1, n2)
            return grades[n1] > grades[n2]
        end)
    end
  ```
    为什么在Lua中允许这种访问？原因在于函数是“第一类值”。考虑以下代码：
  ```lua
    function newCounter()
        local i = 0
        return function()   -- 匿名函数
            i = i + 1
            return i
        end
    end
    
    c1 = newCounter()
    print(c1())      --> 1
    print(c1())      --> 2
  ```
    在这段代码中，匿名函数访问了一个“非局部的变量”i, 该变量用于保存一个计数器。初看上去，由于创建变量i的函数（newCounter）已经返回，所以之后每次调用匿名函数时，i都应是已超出了作用范围的。但其实不然，Lua会以closure的概念来正确地处理这种情况。简单地讲，**一个closure就是一个函数加上该函数所需访问的所有“非局部的变量”**。如果再次调用newCounter，那么它会创建一个新的局部变量i，从而也将得到一个新的closure：
  ```lua
    c2 = newCounter()
    print(c2())      --> 1
    print(c1())      --> 3
    print(c2())      --> 2
  ```
    **因此c1和c2是同一个函数所创建的两个不同的closure，它们各自拥有局部变量i的独立实例**。

    从技术上讲，**Lua中只有closure，而不存在“函数”。因为，函数本身就是一种特殊的closure**。不过只要不会引起混淆，仍将采用术语“函数”来指代closure。

  - **Lua中的函数是存储在普通变量中的，因此可以轻易地重新定义某些函数，甚至是重新定义那些预定义的函数**。
    ```lua
      do
          local oldSin = math.sin
          local k = math.pi/180
          math.sin = function(x)
              return oldSin(x*k)
          end
      end
    ```
      可以使用同样的技术来创建一个安全的运行环境，即所谓的“沙盒”。当执行一些未受信任的代码时就需要一个安全的运行环境，例如在服务器中执行那些从Internet上接受到的代码。举例来说，如果要限制一个程序访问文件的话，只需要使用closure来重定义函数io.open就可以。
    ```lua
      do
          local oldOpen = io.open
          local access_OK = function(filename, mode)
              <检查访问权限>
          end
          io.open = function(filename, mode)
              if access_OK(filename. mode) then
                  return oldOpen(filename, mode)
              else
                  retrun nil, "access denied"
              end
          end
      end
    ```

#### 6.2 非全局的函数
- 在定义递归的局部函数时，还有一个特别之处需要注意。
  ```lua
    local fact = function(n)
        if n == 0 then return 1
        else return n * fact(n-1)   -- 错误
        end
    end
  ```
    当Lua编译到函数体中调用fact(n-1)的地方时，由于局部的fact尚未定义完毕，因此这句表达式其实是调用了一个全局的fact，而非此函数自身。为了解决这个问题，可以先定义一个局部变量，然后再定义函数本身：
  ```lua
    local fact
    fact = function(n)
        if n == 0 then return 1
        else return n * fact(n-1)   
        end
    end
  ```
    当Lua展开局部函数定义的“语法糖”时，并不是使用基本函数定义语法。而是对于局部函数定义：
  ```lua
    local function foo (<参数>) <函数体> end
  ```
    Lua将其展开为：
  ```lua
    local foo
    foo = function (<参数>) <函数体> end
  ```
    因此，使用这种语法来定义递归函数不会产生错误。

#### 6.3 正确的尾调用
当一个函数调用是另一个函数的最后一个动作时，该调用才算是一条“尾调用”。以下代码中对g的调用就是一条“尾调用”：
```lua
function f(x) return g(x) end
```
当f调用完g之后就再无其他事情可做了。因此在这种情况中，程序就不需要返回那个“尾调用”所在的函数了。所以在“尾调用”之后，程序也不需要保存任何关于该函数的栈信息。当g返回时，执行控制权可以直接返回到调用f的那个点上。使得在进行“尾调用”时不耗费任何栈空间。将这种实现称为“尾调用消除”。

**由于“尾调用”不会耗费占空间，所以一个程序可以拥有无数嵌套的“尾调用”**

### 第7章 迭代器与泛型for
#### 7.1 迭代器与closure
迭代器就是一种可以遍历一种集合中所有元素的机制。在Lua中，通常将迭代器表示为函数。每调用一次函数，即返回集合中的“下一个”元素。

作为实例，来为列表编写一个简单的迭代器。与ipairs不同的是该迭代器并不是返回每个元素的索引，而是返回元素的值：
  ```lua
    function values(t)
        local i = 0
        return function() i = i + 1; return t[i] end
    end
  ```
可以在一个while循环中使用这个迭代器：
  ```lua
    t = {10, 20, 30}
    iter = values()
    while true do
        local element = iter()
        if element == nil then break end
        print(element)
    end
  ```
然而使用泛型for则更为简单
  ```lua
    t = {10, 20, 30}
    for element in values(t) do
        print(element)
    end
  ```
泛型for为一次迭代循环做了所有的簿记工作。它在内部保持了迭代器函数，因此不再需要iter变量。**它在每次新迭代时调用迭代器，并在迭代器返回nil时结束循环。**

#### 7.2 泛型for的语义

- 泛型for在循环过程内部保持了迭代器函数。实际上它保存着3个值：一个迭代器函数、一个恒定状态和一个控制变量。
    泛型for的语法如下：
  ```lua
    for <var-list> in <exp-list> do
        <body>
    end
  ```

其中，<var-list>是一个或多个变量名的列表，以逗号分隔；<exp-list>是一个或多个表达式的列表，同样以逗号分隔。通常表达式列表只有一个元素，即一句对迭代器工厂的调用。例如：
  ```lua
    for k, v in pairs(t) do print(k, v) end
  ```
- 在初始化步骤之后，for会以恒定状态和控制变量来调用迭代器函数。然后for将迭代器函数的返回值赋予变量列表中的变量。如果第一个返回值为nil，那么循环终止。否则，for执行它的循环体，随后再次调用迭代器函数，并重复这个过程。
    更明确地说，以下语句：
  ```lua
    for var_1, ..., var_n in <explist> do <block> end
  ```
等价于：
  ```lua
    do
        local _f, _s, _var = <explist>
        while true do
            local var_1, ..., var_n = _f(_s, _var)
            _var = var_1
            if _var == nil then break end
            <block>
        end
    end
  ```

#### 7.3 无状态的迭代器
- 所谓“无状态的迭代器”，就是一种自身不保存任何状态的迭代器。因此，我们可以在多个循环中使用同一个无状态的迭代器，避免创建新的closure开销。例如ipairs

- 当Lua调用for循环中的ipairs(a)时，它会获得3个值：迭代器函数iter、恒定状态a和控制变量的初始值0.然后Lua调用iter(a, 0), 得到1，a[1]。在第二次迭代中，继续调用iter(a, 1), 得到2，a[2], 以此类推，直至得到第一个nil元素为止。

- 在调用next(t, k)时，k时table t的一个key。此调用会以table中任意次序返回一组值：此table的下一个key，及这个key所对应的值。**而调用next(t, nil)时，返回table的第一组值。若没有下一组值时，next返回nil**

#### 7.4 具有复杂状态的迭代器
通常，迭代器需要保存许多状态，可是泛型for却只提供的一个恒定状态和一个控制变量用于状态的保存。一个最简单的解决方法就是使用closure。或还可以将迭代器所需的所有状态打包为一个table，保存在恒定状态中。**一个迭代器通过这个table就可以保存任意多的数据。虽然，在循环过程中恒定状态总是同一个table，但这个table的内容却可以发生改变。**

#### 7.5 真正的迭代器

### 第8章 编译、执行与错误
区别解释性语言的主要特征并不在于是否能编译它们，而是在于编译器是否是运行时库的一部分，即是否有能力（并且轻易地）执行动态生成的代码。

#### 8.1 编译
- loadfile会从一个文件加载Lua代码块，**但它不会运行代码，只是编译代码**，然后将编译结果作为一个函数返回。

#### 8.2 C代码
- loadlib是一个非常底层的函数。必须提供库的完整路径及正确的函数名称。通常使用require来加载C程序库，这个函数会搜索指定的库，然后用loadlib来加载库，并返回初始化函数。这个初始化函数应将库中提供的函数注册到Lua中，就好像一段Lua代码定义了其他的函数一样。


#### 8.3 错误
- 由于想“if not <condition> then error end”这样的组好是非常通用的代码，所以Lua提供了一个内键函数asset来完成此类工作：
  ```lua
    print "enter a number":
    n = asset(io.read("*number"), "invalid input")
  ```
    asset函数检查其第一个参数是否为true。若为true，则简单地返回该函数；否则就引发一个错误。它的第二个参数是一个可选的信息字符串。
- 当一个函数遭遇了一种未预期的状况(既异常), 它可以采取两种基本的行为：返回错误代码（通常是nil）或引发一个错误（调用error）。在这两种选择之间并没有固定的法则，但通常的指导原则是：**易于避免的异常应引发一个错误，否则应返回错误代码。**


#### 8.4 错误处理与异常
- 如果需要在Lua中处理错误，则必须使用函数pcall来包装需要执行的代码。pcall函数会以一种“保护模式”来调用它的第一个参数，因此pcall可以捕获函数执行中的任何错误。如果没有发生错误，pcall会返回true及函数调用的返回值；否则，返回false及错误消息。
  ```lua
    local status, err = pcall(function() error({code = 121}) end)
    print(err.code)      --> 121
  ```

#### 8.5 错误消息与追溯(traceback)
通常在错误发生时，希望得到更多的调试信息，而不是只有发生错误的位置。至少，能够追溯到发生错误时的函数调用情况，显示一个完整的函数调用栈。当pcall返回其错误消息时，它已经销毁了调用栈的部分内容。因此，如果希望得到一个有意义的调用栈，那么就必须在pcall返回前获取该信息。为了达成这一要求，Lua提供了函数**xpcall**。该函数除了接受一个需要被调用的函数之外，还接受第二个参数———— 一个错误处理函数。当发生错误时，Lua会在调用栈展开前调用错误处理函数。于是就可以在这个函数中使用debug库来获取关于错误的额外信息。


### 第9章 协同程序
一个具有多个协同程序的程序在任意时刻只能运行一个协同程序，并且**正在运行的协同程序只会在其显示地要求挂起时，它的执行才会暂停。**

#### 9.1 协同程序基础
- 协同程序的真正强大之处在于函数yield的使用上，该函数可以让一个运行中的协同程序挂起。而之后可以再恢复它的运行。
  ```lua
    co = coroutine.create(function()
        for i=1, 10 do
            print("co", i)
            coroutine.yield()
        end
    end)
  ```
    现在当唤醒这个协同程序时，它就会开始执行，直到第一个yield:
  ```lua
    coroutine.resume(co)    --> co 1
  ```
    如果此时检查其状态，会发现协同程序处于挂起状态，因此可以再次恢复其运行:
  ```lua
    print(coroutine.status(co))  --> suspended
  ```
    从协同程序的角度看，所有在它挂起时发生的活动都发生在yield调用中。当恢复协同程序的执行时，对于yield的调用才最终返回。然后协同程序继续它的执行，直到下一个yield调用或执行结束。

- **请注意，resume 是在保护模式中运行的。因此，如果在一个协同程序的执行中发生任何错误，Lua是不会显示错误消息的，而是将执行权返回给resume调用。**

- 当一个协同程序A唤醒另一个协同程序B时，协同程序A就处于一个特殊状态，既不是挂起状态(无法继续A的执行)，也不是运行状态（是B在运行）。所以将这时的状态称为"正常"状态

- Lua的协同程序还具有一项有用的机制，可以通过一对resume-yield来交换数据。**在第一次调用resume时，并没有对应的yield在等待它，因此所有传递给resume的额外参数都将视为协同程序主函数的参数**：
```lua
  co = coroutine.create(function(a, b, c)
      print("co", a, b, c)
    end)
  coroutine.resume(co, 1, 2, 3)   --> co 1 2 3
```
- 在resume调用返回的内容中，第一个值为true则表示没有错误，而后面所有的值都是对应yield传入的参数：
  ~~~lua
   co = coroutine.create(function(a, b)
        coroutine.yield(a+b, a-b)
      end)
  print(coroutine.resume(co, 20, 10))   --> true 30 10
  ~~~

  与此对应的是，yield返回的额外值就是对应resume传入的参数：
  ~~~lua
    co = coroutine.create(function(
        print("co", coroutine.yield())
    end))
    coroutine.resume(co)
    coroutine.resume(co, 4, 5)   --> co 4, 5
  ~~~
    最后，当一个协同程序结束时，它的主函数所返回的值都将作为对应的resume的返回值：
  ~~~lua
    co = coroutine.create(function()
        return 6, 7
    end)
    print(coroutine.resume(co))   --> true 6 7
  ~~~
#### 9.2 管道与过滤器
#### 9.3 以协同程序实现迭代器
#### 9.4 非抢先式的多线程
- 协同程序与常规的多线程的不同之处在于，协同程序是非抢先式的。当一个协同程序运行时，是无法从外部停止它。只有当协同程序显式地要求挂起时(调用yield)，它才会停止。
- 对于非抢先式的多线程来说，只要有一个线程调用了一个阻塞的操作，整个程序在该操作完成前，都会停止下来。

### 第10章 完整的示例
### 第11章 数据结构
#### 11.1 数组
可以使用0、1或其他任意值来作为数组的起始索引：
~~~lua
-- 使用索引值-5~5来创建一个数组
a = {}
for i=-5, 5 do
	a[i] = 0
end
~~~
#### 11.2 矩阵与多维数组
#### 11.3 链表
#### 11.4 队列与双向队列
#### 11.5 集合与无序组
#### 11.6 字符串缓冲
Java提供了StringBuffer, 在Lua中，我们可以将一个table作为字符串缓冲。其关键是使用函数table.concat
~~~lua
local t = {}
for line in io.lines() do
	t[#t + 1] = line.."\n"
end
local s = table.concat(t)
~~~
先前代码读取同样的文件需要1分钟，而这个实现只需花小于0.5秒的时间。
#### 11.7 图
### 第12章 数据文件与持久性
#### 12.1 数据文件
#### 12.2 串行化
可以使用一种简单且安全的方法来括住一个字符串，那就是以"%q"来使用string.format函数。这样它就会用双引号来括住字符串，并且正确地转移其中的双引号和换行符等其他特殊字符。

Lua5.1 还提供了另一种可以以一种安全的方法来括住任意字符串的方法。这是一种新的标记方式[=[...]=],用于长字符串。通过它就不需要改变任何字符串的内容了。
### 第13章 元表与元方法
在Lua代码中，只能设置table的元表。若要设置其他类型的值的元表，则必须通过c代码来完成。标准的字符串程序库为所有的字符串都设置了一个元表，而其他类型在默认情况中都没有元表。
#### 13.1 算术类的元方法
#### 13.2 关系类的元方法
#### 13.3 库定义的元方法
#### 13.4 table 访问的元方法
##### 13.4.1 __index元方法
当访问一个table中不存在的字段时，得到的结果为nil。实际上，这些访问会促使解释器去查找一个叫__index的元方法。如果没有这个元方法，那么访问结果如前述的为nil。否则，就由这个元方法来提供最终结果。

下面将介绍一个有关继承的典型示例。假设要创建一些描述窗口的table，每个table中必须描述一些窗口参数，例如位置、大写及主题颜色等。所有这些参数都以默认值，因此希望在创建窗口对象时可以仅指定那些不同于默认值的参数。第一种方法是使用一个构造式，在其中填写那些不存在的字段。第二种方法是让新窗口从一个原型窗口处继承所有不存在的字段。首先，声明一个原型和一个构造函数，构造函数创建新的窗口，并使它们共享同一个元表：
~~~lua
Window = {}
Window.prototype = {x=0, y=0, width=100, height=100}
Window.mt = {}
Window.mt.__index = function(table, key)
	return Window.prototype[key]
end
function Window.new(o)
	setmetatable(o, Window.mt)
	return o
end

w = Window.new{x=10, y=20}
print(w.width)   --> 100
~~~

如果不想在访问一个table时涉及到它的__index元方法，可以使用函数rawget，调用rawget(t, i)

###### 13.4.2 __newindex元方法
__newindex元方法与__index类似，不同之处在于__newindex用于table的更新，而__index用于table的查询。当对一个table中不存在的索引赋值时，解释器就会查找__newindex元方法。如果有这个元方法，解释器就调用它，而不是执行赋值。如果这个元方法是一个table，解释器就在此table中执行赋值，而不是对原来的table。有一个原始函数允许绕过元方法：rawset(t, k, v)
##### 13.4.3 具有默认值的table
只需要创建一个新的table，并用它作为key即可 解决table中字段名称冲突
~~~lua
local key = {} -- 唯一的key
local mt = {__index= function(t) return t[key] end}
function setDefault(t, d)
	t[key] = d
	setmetatable(t, mt)
end
~~~
还有一种方法可以将table与其默认值关联起来：使用一个独立的table，它的key为各种table，value就是各种table的默认值。不过，为了正确地实现这种做法，我们还需要一种特殊性质的table，“弱引用table”

##### 13.4.4 跟踪table的访问
__index和__newindex都是在table中没有所需访问的index时才发货作用。因此，只有将一个table保存为空，才有可能捕捉到所有对它的访问。为了监视一个table的所有访问，就应该为真正的table创建一个代理。这个代理就是一个空table，其中__index和__newindex元方法用于跟踪所有的访问，并将访问重定向到原来的table上。

##### 13.4.5 只读的table
~~~lua
function readOnly(t)
	local proxy = {}
	local mt = {
		__index = t,
		__newindex = function(t, k, v)
			error("attemp to update a read-only table", 2)
		end
  	}
  	setmetatable(proxy, mt)
  	return proxy
}
end
~~~

### 第14章 环境
Lua将环境table自身保存在一个全局变量_G中。以下代码打印当前环境中所有的全局变量名称：
~~~lua
for n in pairs(_G) do print(n) end
~~~

#### 14.1 具有动态名字的全局变量
#### 14.2 全局变量声明
#### 14.3 非全局的环境
可以通过函数setfenv来改变一个函数的环境。该函数的参数是一个函数和一个新的环境table。第一个参数除了可以指定为函数本身，还可以指定为一个数字，以表示当前函数调用栈中的层数。数字1表示当前函数，数字2表示调用当前函数的函数。
setfenv(1, {g = _G})

### 第15章 模块与包
#### 15.1 require函数
~~~lua
function require(name)
	if not package.loaded[name] then
		local loader = findloader(name)
		if loader == nil then
			error("unabel to load module"..name)
		end
		package.loaded[name] = true       -- 将模块标记为已加载
		local res = loader(name)          -- 初始化模块
		if res ~= nil then
			package.loaded[name] = res    -- 模块赋值
		end
	end
	return package.loaded[name]
end
~~~

require用于搜索Lua文件的路径存放在变量package.path中
#### 15.2 编写模块的基本方法
#### 15.3 使用环境
#### 15.4 module函数
#### 15.5 子模块与包
Lua使用的目录分隔符是编译时配置的，可以是任意的字符串。例如，在没有目录层级的系统中，就可以使用“_”作为“目录分隔符”。那么require “a.b” 就会搜索到文件 a_b.lua

### 第16章 面向对象编程
使用self参数是所有面向对象语言的一个核心。大多数面向对象语言都能对程序员隐藏部分self参数，从而使得程序员不必显式地声明这个参数。Lua只需使用冒号，则能隐藏该参数。
~~~lua
function Account:withdraw(v)
	self.balance = self.balance - v
end
~~~
等价于
~~~lua
function Account:withdraw(self, v)
	self.balance = self.balance - v
end
~~~
a.withdraw(a, 100)
a:withdraw(100)

#### 16.1 类
#### 16.2 继承
#### 16.3 多重继承
#### 16.4 私密性
通过两个table来表示一个对象。一个table用来保存对象的状态；另一个用于对象的操作。对象本身是通过第二个table来访问的。即通过其接口的方法来访问。为了避免未授权的访问，表示状态的table不保存在其他table中，而只是保存在方法的closure中。（不太常用）
#### 16.5 单一方法做法

### 第17章 弱引用table
Lua的垃圾收集器与一些其他的收集器有所不同，它没有环形引用的问题。当用到环形数据结构时，无须作出任何特殊的处理，它们也可以像其他数据一样被正常回收。

弱引用table就是这样一种机制，用户能用它来告诉Lua一个引用不应该阻碍一个对象的回收。所谓"弱引用"就是一种会被垃圾收集器忽视的对象引用。如果一个对象的所有引用都是弱引用，那么Lua就可以回收这个对象了，并且还可以以某种形式来删除这些弱引用本身。Lua用“弱引用table”来实现“弱引用”，一个弱引用table就是一个具有弱引用条目的table。如果一个对象只被一个弱引用table所持有，那么最终Lua是会回收这个对象的。

Lua有3种弱引用table：具有弱引用key的table、具有弱引用value的table、同时具有两种弱引用table。

一个table的弱引用类型是通过其元表中的--mode字段来决定的。这个字段的值应为一个字符串，如果这个字符串中包含字母'k'， 那么这个table的key是弱引用的；如果这个字符串中包含字母'v',那么这个table的value是弱引用的。
~~~lua
a = {}
b = {__mode="k"}
setmetatable(a, b) -- 现在'a'的key就是弱引用
key = {}		   -- 创建第一个key
a[key] = 1
key = {}		   -- 创建第二个key
a[key] = 2
collectgarbage()   -- 强制进行一次垃圾收集
for k, v in pairs(a) do print(v) end
--> 2
~~~
在本例中， 第二句赋值key={} 会覆盖第一个 key。当收集器运行时，由于没有其他地方在引用第一个key，因此第一个key就被回收了，并且table中的相应条目也被删除了。

主要：Lua只会回收弱引用table中的对象。而像数字和布尔这样的“值”是不可回收的。例如，对于一个插入table的数字key，收集器是永远不会删除它的。当然，如果一个数字key所对应的value被回收了，那么整个条目都会从这个弱引用table中删除。

#### 17.1 备忘录函数
#### 17.2 对象属性
#### 17.3 回顾table的默认值

### 第18章 数学库
### 第19章 table库
### 第20章 字符串库
### 第21章 I/O库
### 第22章 操作系统库
### 第23章 调试库
#### 23.1 自省机制
- debug.getinfo函数，它的第一个参数可以是一个函数或一个栈层。当为某函数foo调用debug.getinfo(foo)时，就会得到一个table，其中包含了一些与该函数相关的信息。

  | 字段              | 含义                           |
  | --------------- | ---------------------------- |
  | source          | 函数定义的位置                      |
  | short_src       | source的短版本，可以用于错误信息中         |
  | linedefined     | 该函数定义在源代码中第一行的行号             |
  | lastlinedefined | 该函数定义在源代码中最后一行的行号            |
  | what            | 函数的类型； Lua函数则为“Lua”；C函数则为“C” |
  | name            | 该函数的一个适当的名称                  |
  | namewhat        | 上一个自动的含义                     |
  | nups            | 该函数的upvalue的数量               |
  | func            | 函数本身                         |

- getinfo函数的效率不高。为了得到更好的性能，getinfo有第二个可选参数，用于指定希望获取哪些信息。这个参数是一个字符串，其中美国字母代码一组字段，这些字母有：
  | 字母   | 获取哪些信息                                   |
  | ---- | ---------------------------------------- |
  | 'n'  | 选择name和namewhat                          |
  | 'f'  | 选择func                                   |
  | 'S'  | 选择source、short_src、what、linedefined和lastlinedefined |
  | '1'  | 选择currentline                            |
  | 'L'  | 选择activelines                            |
  | 'u'  | 选择nups                                   |
  debug.getinfo(1, "S1")

#### 23.2 钩子
- 有4中事件会触发一个钩子：
  - 每当Lua调用一个函数时产生的call事件
  - 每当函数返回时产生的return事件
  - 每当Lua开始执行一行新代码时产生的line事件
  - 当执行完指定数量的指令后产生的count事件
- 若要注册一个钩子，需要用两个或3个参数来调用debug.sethook:第一个参数是钩子函数；第二个参数是一个字符串，描述了需要监控的事件；第三个参数是一个可选数字，用于说明多久获得一次count事件。如要监控call、return和line事件，需要将它们的首字母('c'、'r'、'l')放入掩码字符串。若要监控count事件，则需要在第三个参数中指定一个计数器。若要关闭钩子，只需不带任何参数调用sethook.
  以下代码会打印出解释器执行到的每一行：
   ~~~lua
   debug.sethook(print, "l")
   ~~~
#### 23.3 性能剖析
如果是做计时性的剖析，最好使用C接口，因为每次Lua调用钩子的代价太高，从而使得测试结果偏差较大。

### 第24章 C API概述
### 第25章 扩展应用程序
### 第26章 从Lua调用C
### 第27章 编写C函数的技术
### 第28章 用户自定义类型
### 第29章 管理资源
### 第30章 线程和状态
Lua不支持真正的多线程，也就是不支持那种共享内存的抢先式多线程。有两个原因导致Lua不支持这种多线程。首先，ANSI C没有提供这样的功能，并且也没有可移植的方法能在Lua中实现这种机制。第二个理由则更充分，在Lua中引入多线程并不是一个好选择。

#### 30.1 多个线程
在Lua中，一个线程本质上就是一个协同程序。
从C API的角度看，将线程想象成一个栈可能更形象些。不过从实现的观点来看，一个线程的确就是一个栈。每个栈都保留着一个线程中所有未完成的函数调用信息，这些信息包括调用的函数、每个调用的参数和局部变量。换句话说，一个栈拥有一个线程得以继续运行的所有信息。因为，多个线程就意味着多个独立的栈。

### 第30章 内存管理