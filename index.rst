.. Storm ORM 中文教程 documentation master file, created by
   sphinx-quickstart on Fri May 25 15:16:39 2012.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Storm Tutorial 中文版
==========================================

`Storm <https://storm.canonical.com>`_ 是一个 Python ORM 库，本文是 `Storm 官方教程 <https://storm.canonical.com/Tutorial>`_\ 的简体中文翻译版。

项目 Github 库： `https://github.com/huangz1990/storm_orm_tutorial_cn <https://github.com/huangz1990/storm_orm_tutorial_cn>`_ 。


跑起来再说！
---------------

这份 Storm 教程包含在源代码 ``tests/tutorial.txt`` 当中，因此它会被测试并保持更新。

Ubuntu 的 Storm 包由 Storm 团队提供：

::


    sudo apt-add-repository ppa:storm/ppa
    sudo apt-get update
    sudo apt-get install python-storm


载入
-------

让我们从往命名空间(namespace)载入一些名字(names)开始。


::

    >>> from storm.locals import *
    >>>


基本定义
----------

现在，我们定义一种用几项属性来描述信息的类型(type)，用它来作映射。


    >>> class Person(object):
    ...     __storm_table__ = "person"
    ...     id = Int(primary=True)
    ...     name = Unicode()


注意，这个定义没有使用任何 Storm 定义的 ``base`` 类或构造函数。


创建一个数据库以及仓库(store)
--------------------------------

我们还是什么都没讲到，因此，让我们来定义一个储存在内存中的 SQLite 数据库，以及使用该数据库的仓库来玩玩。

::

    >>> database = create_database("sqlite:")
    >>> store = Store(database)
    >>>

目前，有三种数据库被支持： SQLite ， MySQL 以及 PostgreSQL 。 ``create_database`` 函数接受 URI 作为参数，就像这样：

::

   database = create_database("scheme://username:password@hostname:port/database_name")

其中的 ``scheme`` 可以是 ``sqlite`` ， ``postgres`` ，或 ``mysql`` 。

现在我们要创建一个表，将该表用来保存我们类中的数据。

::

    >>> store.execute("CREATE TABLE person "
    ...               "(id INTEGER PRIMARY KEY, name VARCHAR)")
    <storm.databases.sqlite.SQLiteResult object at 0x...>> 

我们得到了一个结果(result)，不过现在不必关心它。通过使用 ``noresult=True`` ，我们也可以省略所有结果。


创建对象
-----------

让我们通过前面定义的类来创建对象。


::

    >>> joe = Person()
    >>> joe.name = u"Joe Johnes"
    >>>
    >>> print "%r, %r" % (joe.id, joe.name)
    5 None, u'Joe Johnes'

到目前为止，这个对象都还没有连接到数据库。现在，让我们将它加入到前面创建的仓库当中。

::

    >>> store.add(joe)
    <Person object at 0x...>

    >>> print "%r, %r" % (joe.id, joe.name)
    None, u'Joe Johnes'

请注意，这个对象并没有任何改变，即便是被加入到仓库之后 —— 这是因为它还没有刷新呢。


对象的仓库
--------------

一旦对象被加入到仓库，或从仓库中被检索，它和仓库的关系就明了了。我们也可以很容易地验证对象绑定到了哪一个仓库。

::

    >>> Store.of(joe) is store
    True

    >>> Store.of(Person()) is None
    True


对象的查找
------------

现在，如果我们向仓库查询名为 Joe Johnes 的人，会发生什么事？

::

    >>> person = store.find(Person, Person.name == u"Joe Johnes").one()

    >>> print "%r, %r" % (person.id, person.name)
    1, u'Joe Johnes'

Joe Johnes 已经被仓库记录了！是的，就正如你所期待的一样。

我们还可以通过主键(primary key)来检索对象。

::

    >>> store.get(Person, 1).name
    u'Joe Johnes


缓存特性
-----------

有趣的是， ``person`` 变量实际上就是 ``joe`` ，是吧？我们才刚刚添加了一个对象，所以应该只有一个 Joe ，为什么会有两个不同的对象呢？答案是，这两个变量指向的是同一个 Joe ：

::

    >>> person is joe
    True

事实上，每个仓库都一个对象缓存(cache)。当一个对象加入到仓库之后，在它被其他地方引用(reference)期间，仓库会尽可能地缓存它，直到该对象被污染(dirty，即有更改未被刷新)。

Storm 保证至少有一定数目的最近使用对象被保存到内存中用以取代查询，因此频繁使用的对象就不必事无大小都检索数据库了。


刷新
-------

当第一次试图在数据库中查找 Joe 的时候，我们神奇地发现属性 ``id`` 已经被赋值了。这是因为对象已经被隐含地(implicitly)刷新了，也就是说，某些行为也会导致之前的改动操作(changes)生效。

刷新也可以强制地(explicitly)发生：

::

    >>> mary = Person()
    >>> mary.name = u"Mary Margaret"
    >>> store.add(mary)
    <Person object at 0x...>
    >>>
    >>> print "%r, %r" % (mary.id, mary.name)
    None, u'Mary Margaret'
    >>>
    >>> store.flush()
    >>> print "%r, %r" % (mary.id, mary.name)
    2, u'Mary Margaret'


改变仓库中的对象
---------------------

正如往常修改对象一样，我们也可以通过使用表达式修改绑定了数据库的对象，并因此收益。

::

    >>> store.find(Person, Person.name == u"Mary Margaret").set(name=u"Mary Maggie")
    >>> mary.name
    u'Mary Maggie'

此操作会修改数据库及内存中所有匹配对象。


提交
-----

至今为止我们所做的一切都只是事务(transaction)。在这个点上，我们既可以通过提交(committing)操作，将之前未提交的改动操作都执行，又或者，我们可以通过回滚(rolling)操作来取消它们。

让我们来提交它，这很简单：

::

    >>> store.commit()
    >>>

一目了然。一切还是像以前一样，但现在改动操作已经是板上钉钉了。


回滚
------

想要终止改动操作也是同样直观的。

::

    >>> joe.name = u"Tom Thomas"
    >>>

让我们在 Storm 的数据库中看看这种改动将会是什么样子。

::

    >>> person = store.find(Person, Person.name == u"Tom Thomas").one()
    >>> person is joe
    True

一切如常，现在，魔术开始(音乐，起！）。

::

    >>> store.rollback()
    >>>

呃。。。啥事都没发生？

其实不然。 Joe ，他回来了！

::

    >>> print "%r, %r" % (joe.id, joe.name)
    1, u'Joe Johnes'


构造函数
-----------

我们研究人(``Person`` 类)已经够久了。现在，让我们在模型中增加一种新数据：公司(``Company`` 类)。我们将在 ``Company`` 类中使用构造函数，因为这会比较好玩。这将是你所见过的最简单的公司类：

::

    >>> class Company(object):
    ...     __storm_table__ = "company"
    ...     id = Int(primary=True)
    ...     name = Unicode()
    ...
    ...     def __init__(self, name):
    ...         self.name = name

注意，构造函数的参数并非是可选的。如果你愿意，它也可以是可选的，但公司总得有个名字吧。

让我们为它添加表：

::

    >>> store.execute("CREATE TABLE company "
    ...               "(id INTEGER PRIMARY KEY, name VARCHAR)", noresult=True)
    
然后，创建一个新公司：

::

    >>> circus = Company(u"Circus Inc.")

    >>> print "%r, %r" % (circus.id, circus.name)
    None, u'Circus Inc.'
    
    
因为我们还没刷新，所以 ``id`` 仍然是未定义的。事实上，我们甚至还没有把新创建的公司加入到仓库中。我们稍候再来做这事。
    

引用和继承
------------

现在，我们希望指派一些雇员到我们的公司。与其重定义 ``Person`` 类，我们不如让其维持原状，因为它足够一般，我们创建一个它的子类作为雇员类，其中雇员类还有一个新加入的字段：公司的 ``id`` 。

::

    >>> class Employee(Person):
    ...     __storm_table__ = "employee"
    ...     company_id = Int()
    ...     company = Reference(company_id, Company.id)
    ...
    ...     def __init__(self, name):
    ...         self.name = name


注意上面的定义，它没有改写 ``Person`` 类原有的东西，而是引入了引用到其他类的两个属性： ``company_id`` 和 ``company`` 。它也有构造函数，但里面并没有任何关于 ``company`` 的定义。

像往常一样，我们需要一个表。 SQLite 对外键(foreign key)没有太多想法， 所以我们也懒得去定义它。

::

    >>> store.execute("CREATE TABLE employee "
    ...               "(id INTEGER PRIMARY KEY, name VARCHAR, company_id INTEGER)",
    ...               noresult=True)

是时候让 Ben 闪亮登场了：

::

    >>> ben = store.add(Employee(u"Ben Bill"))
    >>>
    >>> print "%r, %r, %r" % (ben.id, ben.name, ben.company_id)
    None, u'Ben Bill', None
    
    
我们可以看到，以上内容还没有被刷新。即便如此，我们也可以断言 Bill 在马戏团(Circus)工作。

::

    >>> ben.company = circus
    >>>
    >>> print "%r, %r" % (ben.company_id, ben.company.name)
    None, u'Circus Inc.'

当然，在数据库刷新之前，我们仍然不知道公司 ``id`` ，也不显式地给他指定一个。即便如此， Storm 还是能保持它们之间的关系。

如果有任何正在等待的操作被刷新到数据库中(无论是隐式或强制)，对象会获取它们的 ``id`` ，并保证在刷新之前，所有引用关系        都可以顺利被更新。

::

    >>> store.flush()
    >>>
    >>> print "%r, %r" % (ben.company_id, ben.company.name)
    1, u'Circus Inc.'

它们都被刷新到数据库里面了。现在，请注意马戏团公司并没有强制地被加入到仓库。Storm 会自动为引用和被引用对象双方完成该任务。

让我们创建另一家公司来检查一下。这一次，我们会在加入新公司之后立刻刷新：

::

    >>> sweets = store.add(Company(u"Sweets Inc."))
    >>> store.flush()
    >>> sweets.id
    2

不错，我们已经得到了新公司的 ``id`` 。那么，如果我们改变 Ben 的 ``company_id`` ，会怎么样？

::

    >>> ben.company_id = 2
    >>> ben.company.name
    u'Sweets Inc.'
    >>> ben.company is sweets
    True

哈哈！没有料到吧？

让我们将所有改动都提交了吧。

::

    >>> store.commit()
    >>>


多对一引用集合
-------------------

我们的模型表示，员工们只在一间公司工作(我们这里只设计平常人)，而公司当然可以有多个员工。在 Storm 中我们用引用集合(reference sets)表示它们。

我们不重新定义 ``Company`` 类，而是将一个新特性(attribute)加给它：

::

    >>> Company.employees = ReferenceSet(Company.id, Employee.company_id)

仅次而已，我们已经可以查看哪些员工在给定的公司工作。

::

    >>> sweets.employees.count()
    1

    >>> for employee in sweets.employees:
    ...     print "%r, %r" % (employee.id, employee.name)
    ...     print employee is ben
    ...
    1, u'Ben Bill'
    True

让我们创建另一名雇员，并将他加入到公司当中：

::

    >>> mike = store.add(Employee(u"Mike Mayer"))
    >>> sweets.employees.add(mike)

毋庸置疑，Mike 现在已经是公司的职员了，这一情况也应该被反映在其他地方。

::

    >>> mike.company_id
    2

    >>> mike.company is sweets
    True


多对多引用集合以及组合键(composed keys)
----------------------------------------------

我们还打算在模型中表示会计师。公司都有会计师，但会计师通常可以为多间公司服务，因此我们用多对多关系来表示它们。

让我们创建简单的类，用来表示会计师及其关系。

::

    >>> class Accountant(Person):
    ...     __storm_table__ = "accountant"
    ...     def __init__(self, name):
    ...         self.name = name
    
    >>> class CompanyAccountant(object):
    ...     __storm_table__ = "company_accountant"
    ...     __storm_primary__ = "company_id", "accountant_id"
    ...     company_id = Int()
    ...     accountant_id = Int()
    

嘿，我们刚刚定义了带组合键的类！

现在，让我们使用它在 ``Company`` 类中定义多对多关系。再一次，我们将新属性塞进已有的对象当中。在编写类的时候定义这些属性也很简单。稍候我们会看到如何用别的方法做这事。

::

    >>> Company.accountants = ReferenceSet(Company.id,
    ...                                    CompanyAccountant.company_id,
    ...                                    CompanyAccountant.accountant_id,
    ...                                    Accountant.id)

搞定！属性排列的先后顺序事关重大，但这里面的逻辑应该是一目了然的。

在这个点上，我们还缺一些表。

::

    >>> store.execute("CREATE TABLE accountant "
    ...               "(id INTEGER PRIMARY KEY, name VARCHAR)", noresult=True)
    ...

    >>> store.execute("CREATE TABLE company_accountant "
    ...               "(company_id INTEGER, accountant_id INTEGER,"
    ...               " PRIMARY KEY (company_id, accountant_id))", noresult=True)
    
我们将一对会计夫妇登记到两个公司。

::

    >>> karl = Accountant(u"Karl Kent")
    >>> frank = Accountant(u"Frank Fourt")

    >>> sweets.accountants.add(karl)
    >>> sweets.accountants.add(frank)

    >>> circus.accountants.add(frank)
    >>>
    
大功告成！真的！请注意，我们甚至不必将它们添加到仓库，当它们链接到其他已经在仓库内的对象时，这已经被隐式地执行了，我们也无须宣告对象间的关系，因为那已经在引用对象集合中明晰了。

现在，让我们来验证一下。

::

    >>> sweets.accountants.count()
    2

    >>> circus.accountants.count()
    1

我们还没有显式地使用过 ``CompanyAccountant`` 对象，假如你好奇的话，也可以验证一下它。

::

    >>> store.get(CompanyAccountant, (sweets.id, frank.id))
    <CompanyAccountant object at 0x...>> 


请注意，因为组合键的关系，我们往 ``get`` 方法传递了一个元组(tuple)。

如果我们想知道会计师为哪些公司工作， 我们可以很容易地定义一个反向(reversed)关系：

::

    >>> Accountant.companies = ReferenceSet(Accountant.id,
    ...                                     CompanyAccountant.accountant_id,
    ...                                     CompanyAccountant.company_id,
    ...                                     Company.id)

    >>> [company.name for company in frank.companies]
    [u'Circus Inc.', u'Sweets Inc.']

    >>> [company.name for company in karl.companies]
    [u'Sweets Inc.']


联接(Joins)
-------------

既然已经有了一些不错的数据，让我们尝试做一些有趣的查询(queries)玩玩看。

先来检查看看哪些公司至少有一名叫 Ben 的员工。我们至少有两种方法可以做到这一点。

首先，使用隐式联接。

::

    >>> result = store.find(Company,
    ...                     Employee.company_id == Company.id,
    ...                     Employee.name.like(u"Ben %"))
    ...

    >>> [company.name for company in result]
    [u'Sweets Inc.']

然后，我们也可以做一个显式联接。这是 Storm 查询当中，映射复杂 SQL 联接的有趣之处。

::

    >>> origin = [Company, Join(Employee, Employee.company_id == Company.id)]
    >>> result = store.using(*origin).find(Company, Employee.name.like(u"Ben %"))

    >>> [company.name for company in result]
    [u'Sweets Inc.']

如果我们已经定义过公司，并且想知道公司内哪个员工名叫 Ben ，这再容易不过了。

::

    >>> result = sweets.employees.find(Employee.name.like(u"Ben %"))

    >>> [employee.name for employee in result]
    [u'Ben Bill']


子查询
--------

假设我们想找出所有不属于公司的会计，可以使用子查询。

::

    >>> laura = Accountant(u"Laura Montgomery")
    >>> store.add(laura)
    <Accountant ...>

    >>> subselect = Select(CompanyAccountant.accountant_id, distinct=True)
    >>> result = store.find(Accountant, Not(Accountant.id.is_in(subselect)))
    >>> result.one() is laura
    True
    

排序以及限制(limiting)结果
----------------------------

排序和限制结果通常是这样那样工具的最简单而又最迫切的功能，所以我们希望它们能够既简单又易用，这是当然的了。

一行代码胜过千言万语，这里有几个例子，说明它是如何工作的：

::

    >>> garry = store.add(Employee(u"Garry Glare"))

    >>> result = store.find(Employee)

    >>> [employee.name for employee in result.order_by(Employee.name)]
    [u'Ben Bill', u'Garry Glare', u'Mike Mayer']

    >>> [employee.name for employee in result.order_by(Desc(Employee.name))]
    [u'Mike Mayer', u'Garry Glare', u'Ben Bill']

    >>> [employee.name for employee in result.order_by(Employee.name)[:2]]
    [u'Ben Bill', u'Garry Glare']


多类型查询
------------

有时候，用一个查询检索多于一个对象也是蛮有趣的。想象一下，例如除了想知道哪个公司有名叫 Ben 的员工外，我们也想知道这名员工到底是何许人也。这可以用类似如下的查询来完成：

::

    >>> result = store.find((Company, Employee),
    ...                     Employee.company_id == Company.id,
    ...                     Employee.name.like(u"Ben %"))

    >>> [(company.name, employee.name) for company, employee in result]
    [(u'Sweets Inc.', u'Ben Bill')]
    
    
Storm基类
-------------

到目前为止，我们已经用类和类的属性定义好了引用关系及引用集合。这一做法有一定优势，比如更易于调试，但也有一定缺点，比如因为类需要在本地范围(local scope)内被表示(be present)，可能会导致循环引用(circular import)情况。

为了避免那样的状况，Storm 支持使用字符串化的方式，为类和属性名字(names)定义引用关系。这样做的唯一不便是，所有相关的类都必须继承 Storm 类。

让我们定义一些新的类来演示一下。为了揭示要点，我们将在类实际定义引用关系之前引用它们。

::

    >>> class Country(Storm):
    ...     __storm_table__ = "country"
    ...     id = Int(primary=True)
    ...     name = Unicode()
    ...     currency_id = Int()
    ...     currency = Reference(currency_id, "Currency.id")
    
    >>> class Currency(Storm):
    ...     __storm_table__ = "currency"
    ...     id = Int(primary=True)
    ...     symbol = Unicode()

    >>> store.execute("CREATE TABLE country "
    ...               "(id INTEGER PRIMARY KEY, name VARCHAR, currency_id INTEGER)",
    ...               noresult=True)
    
    >>> store.execute("CREATE TABLE currency "
    ...               "(id INTEGER PRIMARY KEY, symbol VARCHAR)", noresult=True)
    
    
现在，让我们看看它是否工作：

::

    >>> real = store.add(Currency())
    >>> real.id = 1
    >>> real.symbol = u"BRL"

    >>> brazil = store.add(Country())
    >>> brazil.name = u"Brazil"
    >>> brazil.currency_id = 1

    >>> brazil.currency.symbol
    u'BRL'


载入钩子
-------------

Storm 允许类定义几个不同的钩子，当某些事情发生时，它们就会采取行动。这里就有一个有趣的钩子： ``__storm_loaded__`` 。

让我们定义 ``Person`` 的一个临时的子类，然后来测试一下它。

::

    >>> class PersonWithHook(Person):
    ...     def __init__(self, name):
    ...         print "Creating %s" % name
    ...         self.name = name
    ...
    ...     def __storm_loaded__(self):
    ...         print "Loaded %s" % self.name

    >>> earl = store.add(PersonWithHook(u"Earl Easton"))
    Creating Earl Easton

    >>> earl = store.find(PersonWithHook, name=u"Earl Easton").one()

    >>> store.invalidate(earl)
    >>> del earl
    >>> import gc
    >>> collected = gc.collect()

    >>> earl = store.find(PersonWithHook, name=u"Earl Easton").one()
    Loaded Earl Easton
    
    
请注意，在第一次查找(find)的时候，没有返回结果，因为当时对象还只存在于内存和缓存中。然后，我们从 Storm 缓存中无效化(invalidated)对象，并触发垃圾回收以确保对象不在内存当中。在此之后，对象必须通过数据库被再次检索，钩子(不是构造函数！)也因此被调用。


执行表达式
-------------

Storm 还提供了一种非数据库式(database-agnostic way)的方式来执行表达式，以备不时之需。
例如：

::

    >>> result = store.execute(Select(Person.name, Person.id == 1))
    >>> result.get_one()
    3 (u'Joe Johnes',)

这一机制被 Storm 自身内部用以实现更高级功能。


自动重载(reloading)值
------------------------

Storm 提供了一些在它控制之下的特殊值，可以将它们赋给属性。其中一个特殊值就是 ``AutoReload`` 。启用它后，每次接触(touched)数据库，对象的值都将被自动重载。这对主键可能有用，就如下面的例子所示。

::

    >>> from storm.locals import AutoReload

    >>> ruy = store.add(Person())
    >>> ruy.name = u"Ruy"
    >>> print ruy.id
    None

    >>> ruy.id = AutoReload
    >>> print ruy.id
    4

当有需要时，可以将属性的默认值设为 ``AutoReload`` ，让对象实行自动刷新。


表达式的值
-----------------

可以赋给属性的除了自动重载之外，还有“惰性表达式”。这种表达式只在属性被访问、或者当对象被刷新到数据库(插入/更新)时，才会被刷新到数据库。

例如：

::

    from storm.locals import SQL

    >>> ruy.name = SQL("(SELECT name || ? FROM person WHERE id=4)", (" Ritcher",))
    >>> ruy.name
    u'Ruy Ritcher'
    
    
请注意，这只是一个关于“可以做什么”的例子，并不是说非要这样写 SQL 语句不可。你可以用 Storm 提供的基于类的 SQL 表达式，把惰性表达式忘到九霄云外。


别名(aliase)
------------------

现在，我们想要找出所有在同一家公司工作的人——我不知道到底是谁想这么干，但这是一个使用别名的好机会。

首先，我们要将 ``ClassAlias`` 引入到本地命名空间当中(友情提示： ``storm.local`` 也应该被载入)，并且创建一个引用指向它。

::

    >>> from storm.info import ClassAlias
    >>> AnotherEmployee = ClassAlias(Employee)

它看上去不错，不是嘛？

现在，我们可以用一种简单直接的方式来查询：

::

    >>> result = store.find((Employee, AnotherEmployee),
    ...                     Employee.company_id == AnotherEmployee.company_id,
    ...                     Employee.id > AnotherEmployee.id)

    >>> for employee1, employee2 in result:
    ...     print (employee1.name, employee2.name)
    (u'Mike Mayer', u'Ben Bill')
    
    
哇！ Mike 和 Ben 为同一家公司工作！

(考考你：为什么上面的查询里要使用大于号？)


调试
--------

某些时候，你需查看 Storm 正在执行的语句。一个建立在 Storm 跟踪系统之上的调试跟踪器可以用来查看引擎盖下到底发生了什么事。跟踪器是一个对象，当有趣的事情发生——比如当 Storm 执行一个语句时，它会收到通知。提供了一个函数来开启和关闭语句跟踪语句。语句默认记录在 ``sys.stderr`` 之下，也可以自己指定一个流。

::

    >>> import sys
    >>> from storm.tracer import debug

    >>> debug(True, stream=sys.stdout)
    >>> result = store.find((Employee, AnotherEmployee),
    ...                     Employee.company_id == AnotherEmployee.company_id,
    ...                     Employee.id > AnotherEmployee.id)
    >>> list(result)
    EXECUTE: 'SELECT employee.company_id, employee.id, employee.name, "...".company_id, "...".id, "...".name FROM employee, employee AS "..." WHERE employee.company_id = "...".company_id AND employee.id > "...".id', ()
    [(<Employee object at ...>, <Employee object at ...>)]

    >>> debug(False)
    >>> list(result)
    [(<Employee object at ...>, <Employee object at ...>)]
    
    
更多！
---------

关于 Storm 还有很多值得谈谈。这个教程只提供了一些入门概念。如果你的问题在别的地方找不到答案，欢迎你到邮件列表里发问。
