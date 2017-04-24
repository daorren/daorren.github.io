---
title: 《Ruby设计模式》笔记
date: 2017-03-31 21:41:23
tags: Ruby 读书笔记
---
## 第一部分 设计模式与Ruby
  - 把变的和不变的分开
  - 针对接口编程，而不对实现编程
    在Ruby中，虽然没有接口这种实际结构，但其真实意义是尽可能针对最通用的类型进行编程。
    如果能用交通工具来处理汽车类型，就别对汽车使用汽车类型。
  - 组合优先于继承
    使用继承，一旦需要修改超类的行为，子类都会受到影响。所以使用组合进行装配会更好。
    所谓组合，就是在实例中，用实例变量保存需要的对象。
    ```ruby
    class Engine
      def start；end
    end
    class Car
      def initialize
        @engine = Engine.new
      end
      def drive
        @engine.start
      end
    end
    ```
  - 委托，委托，委托
    Car类的这个方法就是委托。组合和委托并用是比继承更强大灵活的手段，且没有继承的副作用。代价是对性能的轻微影响，以及需要编写模板代码。
    ```ruby
    def start_engine
      @engine.start
    end
    ```

<!-- more -->

## 第二部分 Ruby中的经典模式
  - 模板方法：变换算法中的一部分
    创建一个包含骨架方法的基类。这个骨架方法（模板方法）驱动了需要变化的部分，但它只是通过调用其他抽象方法（具体实现由子类提供）来实现。
    而由于基于继承，所以短板也会在这里。
    其实思想也是分开变与不变。模板方法中的基本算法是不变部分，子类中的实现是变动部分。
  - 策略：替换算法
    把算法提取出来放到一个对象中，实际使用时调用这个对象（或子类）。
    策略的使用者（环境类）不再需要知道策略的细节，基于组合和委托，切换策略也会很简单，`report.formatter = AnotherFormatter.new`。
    和模板方法都是决定在一些地方使用哪种算法的变化，只不过一个是基于继承，一个基于组合和继承。
    附加：共享数据时：把需要的东西以参数传入；直接把自己 self 传入。
    ```ruby
    class Report
      attr_accessor :formatter
      def initialize(formatter)
        @formatter = formatter
      end
      def output_report
        @formatter.output_report(self) # @formatter.output_report(@title, @text)
      end
    end
    # class Formatter; end // 它本来应该定义格式化接口。但是基于鸭子类型，子类实现相同接口，也没必要这个什么都不做的超类了
    class HTMLFormatter # < Formatter
      def output_report; end
    end
    Report.new(HTMLFormatter.new).output_report
    ```
    由于策略对象是一群被包裹在对象中并知道自己要干什么的可执行代码，而这也正是Ruby中Proc对象的描述。下面是重构。不再需要新类
    ```ruby
    def output_report
      @formatter.call(self)
    end
    HTML_Formatter = lambda do
    end
    Report.new(&HTML_Formatter).output_report
    ```
  - 观察器模式
    由于对象信息变更而产生的消息来源，和消息的消费者之间，建立干净的接口的方法。
    ```ruby
    module Subject
      def initialize
        @observers = []
      end
      def add_observer;end
      def remove_observer;end
      def notify_observers
        @observers.each do |observer|
          observer.update(self)
        end
      end
    end
    class Employee
      include Subject
      def initialize
        super()
      end
      def salary=(new_salary)
        @salary = new_salary
        notify_observers
      end
    end
    ```
  - 组合模式：将部分组合成整体
    有些复杂对象，看起来和构造它们的简单对象一样，或者我们希望它们的行为差不多。
    组合模式使我们可以构建任意复杂的对象树，一个组合结点下可能是另一个组合结点或者只是叶子结点。
    ```ruby
    class Task # 通用接口或基类，称为Component组件。
      def initialize(name)
        @name = name
      end
      def get_time_required;end
    end
    class MixTask < Task # 作为简单对象的leaf叶子类
      def initialize
        super('Mix that batter up')
      end
      def get_time_required;  3.0;  end
    end
    def MakeBatterTask < CompositeTask # 高端类称为composite组合类。同时他自身也是叶子类，构造更高层的组合类。
      def initialize
        super('Make Batter')
        add_sub_task(MixTask.new)
      end
    end
    class CompositeTask < Task # 组合任务中对子类的管理，放入另一个基类
      def initialize(name)
        super(name)
        @sub_tasks = []
      end
      def add_sub_task(task)
        @sub_tasks << task
      end
      def get_time_required
        time = 0.0
        @sub_tasks.each { |t| time += t.get_time_required}
        time
      end
    end
    ```
  - 迭代器模式：遍历集合
    提供一种以依次访问聚合对象而不暴露内部实现结构的途径。
    外部迭代器：迭代器是不同于聚合对象的独立对象，在Ruby中很少使用。其优势是迭代器在外部，所以操作对象灵活性很大。
    内部对象：所有迭代都发生在聚合对象内部，通过使用代码块，就可以将逻辑传入一个聚合。其优势是代码简单清晰。
    ```Ruby
    def for_each_element(array)
      i = 0
      while i < array.length
        yield(array[i])
        i += 1
      end
    end
    ```
    混入Enumerable模块。实现内部迭代器each方法，以及< = >这样的比较操作符，即可混入。它能带来一系列有用的方法。
  - 命令模式：完成任务
    将具体的东西提取出来，放入它自己的对象中的方法，就是命令模式的精华。
    命令就是对知道做什么的代码的封装，它唯一的功能就是等待被执行。Proc也是一段封装一段代码的对象，所以如果任务简单，直接用Proc代替命令类也是不错的。
    这个模式将变动的东西，从不变的东西中分离出来。比如按钮按下你希望发生的事情（变动的东西），和框架提供的标准按钮（不变的东西）。它们之间的关联只存在于运行时，所以很容易改变关联。
    Rails Migration其实就是个命令模式，执行就跑up方法，回退就跑down方法。
  - 适配器模式：填补空隙
    适配器，就是用来跨越在你具有的接口和你需要的接口之间的对象。
    另一种选择，其实不需要新建适配器类，也能通过直接创建原目标对象的接口的方法，达到目的。（修改目标类或者只是目标对象实例）
    适配器模式是我们遇到的第一个系列模式，系列模式指的是，一个对象包含在另一个对象中。这种替身系列还有代理和装饰器。
  - 代理模式：来到对象面前
    代理模式，常用于控制对象的访问权限、提供和位置无关的对象访问途径、延迟对象的创建等。将非核心工作从主题对象中分离出来了。
    代理对象内隐藏着对真实对象的引用，当调用代理对象的方法时，这个代理将请求转发到真实对象。
    - 保护代理：访问权限控制
      ```ruby
      def deposit # in Proxy class
        check_access
        @account.deposit
      end
      ```
    - 远程代理：对象访问途径
      远程代理是一个在客户计算机中的，从客户的代码中看上去和真实目标对象相同的对象。远程代理收到调用请求，它会将请求打包，发送到网络另一端，等待回复，将结果返回。实际上，这就是所有远程过程调用（RPC）系统的工作原理。
      下面是Ruby的SOAP客户机制，创建一个代理。一旦创建后，客户代码不需要担心这个服务事实上位于 www.webservicex.net。
      ```ruby
      require "soap/wsdlDriver"
      wsdl_url = 'http://www.webservicex.net/WeatherForcast.asmx?WSDL'
      proxy = SOAP::WSDLDriverFactory.new(wsdl_url).create_rpc_driver
      weather_info = proxy.GetWeatherByZipCode('ZipCode' => '19128')
      ```
    - 虚拟代理：延迟对象创建
      假装自己是真实对象，而直到调用一个方法前，它甚至还不具备指向该真实对象的引用。
      ```ruby
      def deposit
        s = subject
        return s.deposit
      end
      def subject
        @subject ||= Account.new # 这里将主题对象混合进来了不太好。可以用Proc来改善。
      end
      ```
    无痛代理：我们不可能为目标对象的所有方法都创建代理方法。通过 method_missing，可以对无需修改的对象方法进行无痛代理。
    method_missing常见问题：对象会有一组从Object继承来的基本方法，比如to_s。这个调用不会走到method_missing里面来。
  - 装饰器：改善对象
    由来
      设想写入文本，有时候要加上行号，有时候需要加上时间戳，有时候需要统计字数。可能你会分别构造三个带场景的写入方法，那你就完全误入歧途。
      首先每个使用者都必须知道它到底要写入什么，而且不只是一开始、每次写入都需要知道。一旦犯错就出很大的问题。另一个问题，“把一切放入一个类”，迟早会出问题。
      如果使用基于继承的子类的做法，那么你需要在设计时考虑所有的组合。很多时候你不需要每个组合，你只需要你需要的。
      所以更好的方案是，运行时建立功能组合。一个普通的文本写入类，如果你需要行编号，只要在这个写入类和客户之间插入一个对象，让它给行编号即可。
      这个中间对象把自己贡献给了写入类的功能，也就是它装饰了写入类。这就是装饰器模式名字的由来。
    装饰器类其实有点像代理类，实现了真实对象的所有接口，只是中间加上了他自身的功能。其内部也不真实拥有真实类的实例，所以其实可以是任何装饰器，意味着理论上可以构造任意长的装饰链。链上的每个环节秘密地添加一点功能，达到了功能组合的目的。
      `writer = TimestampingWriter.new(NumberingWriter.new(SimpleWriter.new('final.txt')))`
    减轻代理的郁闷（用于装饰器的基类）：使用 def_delegators 进行代理，它比 method_missing 更为精准。
    Ruby为装饰器带来新的东西
    - 使用方法别名风格的对象进行装饰
      ```ruby
      alias old_write_line write_line
      def write_line(line)
        old_write_line("#{Time.new}: #{line}")
      end
      ```
      在小规模问题当中，十分管用，值得被每个Ruby工程师收录到工具盒中。
    - 使用模组进行装饰
      把之前的装饰类修改为模组。extend方法本质上，是将一个模组插入到对象的继承树中的标准类之前。
      ```ruby
      w = SimpleWriter.new('out')
      w.extend(NumberingWriter)
      ```
    总结：你只需要一个包含最基本功能的类和一系列配合的装饰器。每个装饰器支持相同的核心接口，并将自己的修改加入接口中。它为那些庞大的、“包含你所需要的一切”的对象的方案给出另一种途径。
    **装饰器是本书中最后一个“一个对象包含另一个对象”的模式，另两个是适配器（用正确的接口来包裹，隐藏错误接口）和代理（它的存在不是为了翻译，而是为了控制，适用于之前提到的三种情况）。装饰器，则让你把功能层叠到一个基本对象上**。
  - 单例：只有一个
    单例是一种只能具有一个实例的类，而且该实例能够提供全局访问。
    ```ruby
    class SimpleLogger
      @@instance = SimpleLogger.new
      def self.instance
        return @@instance
      end
      private_class_method :new
    end
    ```
    或者直接使用Ruby的Singleton模组。只要include进来，它为我们做了那些繁重的工作。另外，它是直到调用instance方法，才创建这个实例。
    其他单例模式
    - 全局变量。
      尽管以 $ 开头的变量是全局变量，但它可以被重新赋值。常量也可以作为备选。
      但实际上，这两种都不能阻止人继续创建实例，所以不是很合适。
    - 使用类作为单例。每个类是唯一的，可将单例定义为一个类对象的方法和变量。惰性初始化不太好，类被装载时就已经被初始化了。
    - 使用模组作为单例。
  - 工厂模式：挑选正确的类
    把“哪个类”的决定放到子类中的技巧，称为工厂模式。
    实际上工厂模式不是一种新模式，它是模板方法模式在创建新对象问题上的特殊应用。
    专门用来创建能共存的对象组的对象，叫抽象工厂。
    Ruby中，推算类很容易，比如使用 `const_get` 方法。
  - 生成器：简化对象创建
    当你的对象很难创建、子对象分散在多处时，你可以使用一个独立的类来重构这些创建代码，那个类就是生成器。
    重用生成器：重用生成器实例，也是个不错的选择。而不是每次需要新目标对象的时候，都要构造一个新的生成器。（对象越多程序越慢）
    魔术方法构造更好的生成器：根据特定模式来构造方法名字，如 `add_dvd_and_harddisk`，在 `method_missing` 中识别该模式，并加入逻辑。
  - 解释器：组建系统
    某些时候，解决问题的最好办法是，发明一门专门用于处理那个问题的语言。
    一般先使用分析器构造AST，事实上抽象语法树是解释器模式的核心。想象数学算式的拆解。
    解释器不关心AST的由来，而是应用AST。开始具体的解释AST的时候，或与非这样的逻辑运算往往是常见的操作。

## 第三部分 Ruby新发展的设计模式
  - DSL域指定语言：打开系统
    如果说解释器不关心AST的由来，DSL模式就是关心语言本身而不是解释器。
    核心是eval，将DSL当做Ruby代码来运行。
  - 元编程创建自定义对象
    - 在方法内部，根据条件再次定义方法
    - 在方法内部，根据条件extend模块。
    - 召唤新方法
      ```ruby
      def self.member_of(name)
        code = %Q{
          attr_accessor :parent_#{name}
        }
        class_eval(code)
      end
      ```
      在当前class中对字符串进行求值。其实根据这个方法，容易猜到 attr_accessor 大概也是这样的，它并不是Ruby中的关键字，只是不普通方法。
      ```Ruby
      def self.readable_attr(name)
        code = %Q{
          def #{name}
            @#{name}
          end
        }
        class_eval(code)
      end
      ```
  - 惯例优于配置
    这被认为是Rails大获成功的关键因素之一。
    例子：适配器类命名为<name>Adapter，并保存在adapter目录下，这样对于程序来说，动态使用类、加载文件都比较方便。
