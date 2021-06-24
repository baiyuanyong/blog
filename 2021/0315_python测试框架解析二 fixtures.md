# fixtures

fixture代表了测试中需要使用的资源，需要在使用前创建并在使用后销毁。在unittest中，这样的资源通常是在setUp中建立并在teardown中销毁，但是如果资源需要在多个用例中共享或者比较复杂的情况下，使用fixtures会更加方便一些，而且fixtures本身预置了一些组件，可以直接使用。

下面我们分析一下fixtures的源码结构，比较简单，看下TestWithFixtures精简后的源码

```python
class TestWithFixtures(unittest.TestCase):
    def useFixture(self, fixture):
        self._clear_cleanups()
        try:
            fixture.setUp()
        else:
            self.addCleanup(fixture.cleanUp) 
            #此处调用的是TestCase.addCleanup方法进行注册，而不是fixture的addCleanup方法
            return fixture

class Fixtures:
    def _clear_cleanups(self):
        self._cleanups = CallMany()
        #代理多个cleanup的方法调用
        
class CallMany(object):
    """A stack of functions which will all be called on __call__.
    CallMany also acts as a context manager for convenience.
    Functions are called in last pushed first executed order.
    This is used by Fixture to manage its addCleanup feature.
    """
    def __call__(self, raise_errors=True):
        ......
        #依次调用注册的清理方法
        for cleanup, args, kwargs in cleanups:
            try:
                cleanup(*args, **kwargs)
            ......
```

通过调用useFixture引入资源，首先会使用_clear_cleanups进行初始化，在这里\_cleanups用了一个代理类CallMany而不是一个简单的列表，用于对同一方法的多个实现进行调用，代理类的使用在nose，pytest中均有用到，只不过是内部的实现不同。

然后是使用setUp初始化，底层调用了\_setUp，这也是为什么子类实现一般都是重写\_setUp而不是setUp，setUp在初始化出错的时候会进行一些清理操作，最好不要直接重写覆盖。

```python
    def setUp(self):
        """Prepare the Fixture for use.

        This should not be overridden. Concrete fixtures should implement
        _setUp. Overriding of setUp is still supported, just not recommended.

        After setUp has completed, the fixture will have one or more attributes
        which can be used (these depend totally on the concrete subclass).
        """
        self._clear_cleanups()
        try:
            self._setUp()
        except:
            ......
            errors = [err] + self.cleanUp(raise_first=False)
            ......
```

销毁fixture有两种方法，一种是使用fixture自己的addCleanup将方法保存到自己的\_cleanups这个CallMany代理中，cleanUp使用默认实现，调用CallMany()做销毁工作；一种是重新实现cleanUp方法，用自己的方法替换。不管用什么样的方法，最终在useFixture中会使用unittest.TestCase.addCleanup添加cleanUp到testcase内部\_cleanups队列中，用于后续的清理时调用。

回看unittest.TestCase.run方法，测试完成后总会调用doCleanups，会把之前保存在\_cleanups队列中的方法依次执行，也就是调用fixture的cleanUp自定义方法或者是CallMany代理的多个清理动作。

```python
class TestCase:
    def run(self, result=None):
        ......
        try:
            self._outcome = outcome

            with outcome.testPartExecutor(self):
                self.setUp()
            if outcome.success:
                outcome.expecting_failure = expecting_failure
                with outcome.testPartExecutor(self, isTest=True):
                    testMethod()
                outcome.expecting_failure = False
                with outcome.testPartExecutor(self):
                    self.tearDown()

            self.doCleanups()
            ......

    def doCleanups(self):
        while self._cleanups:
            function, args, kwargs = self._cleanups.pop()
            with outcome.testPartExecutor(self):
                function(*args, **kwargs)
```

作为测试框架的第二部分，理解fixture还是挺重要的，也为后续的pytest分析做一个准备，pytest用到了大量的fixture，也有不同的使用方式。

示例，用两种方法实现资源销毁

```python
import fixtures

class MyFixture1(fixtures.Fixture):
    def _setUp(self):
        self.testa = 42
        self.addCleanup(self._cleanUp)
        print("MyFixture1.setup()")
        
    def _cleanUp(self):
        print("MyFixture1.cleanup()")

class MyFixture2(fixtures.Fixture):
    def _setUp(self):
        self.testb = 42
        print("MyFixture2.setup()")
        
    def cleanUp(self):
        print("MyFixture2.cleanup()")
           
class MyTestCase(fixtures.testcase.TestWithFixtures):
           
    def setUp(self):
        self.fixture1 = self.useFixture(MyFixture1())
        self.fixture2 = self.useFixture(MyFixture2())

    def tearDown(self):
        print("tearDown")
        
    def test_case_1(self):
        self.assertEqual(42, self.fixture1.testa)
        print('exec teting')
        
'''-----------------        
#输出：
MyFixture1.setup()
MyFixture2.setup()
exec teting
tearDown
MyFixture2.cleanup()
MyFixture1.cleanup()
#----------------'''

```

