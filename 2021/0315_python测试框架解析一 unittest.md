# unittest

unittest对于很多人来说都不陌生，如何使用在网上已经有大把的文章介绍，这里就不再进行详细的说明。接下来，让我用一系列的文章来对测试框架实现进行剖析，让大家了解到每个框架是如何实现的，内部逻辑是什么样的，同时我们也用尽可能简短的代码来实现一个类似的功能，管中窥豹，方便在了解原理的基础上进行二次开发。

## 5大组件

- TestCase：测试用例的基类

- TestSuite：用于组合测试用例

- TestRunner：运行测试用例

- TestResult：收集测试报告

- TestLoader：根据参数，一般是命令行参数，解析测试用例并生成TestSuite

记住这几个组件的功能，后续其它的框架分析中我们会看到，基本都是围绕着这几个组件来进行改进。

## 流程分析

运行一个测试，调用方法如下

```python
if __name__ == '__main__':
  unittest.main()
```

main()实际是通过TestProgram这个类完成测试。简单的函数调用解释如下，可以看到5个组件都已经包含在内了，这个顺序在后续的其它框架解析中我们也会看到，基本是相同的：参数添加，解析配置，发现用例，运行用例。

```python
class TestProgram(object):
    def __init__():
        #解析并加载testSuite
        self.parseArgs(argv)
            #初始化argparser，并对参数进行解析
            self._initArgParsers()
            self._main_parser.parse_args(argv[1:], self)
            #使用testLoader发现并把testcase加载到testsuite中
            self.createTests()
                self.testLoader.loadTestsFromModule
        #运行测试
        self.runTests()
            #初始testrunner，其中testResult在内部初始化完成
            testRunner = runner.TextTestRunner()
            testRunner.run(self.test)
```

接下来，让我们用一个极简化的代码来实现一个基本功能的测试框架**minitest**，可以尝试运行并修改。这些代码基本上都是从unittest精简而来，用来学习了解框架如何运行。

```python
import sys
import traceback
import time


class TestLoader(object):
    testMethodPrefix = 'test_'

    '''用来识别test_开头的用例，返回一个TestSuite'''
    def loadTestsFromModule(self, module):
        tests = []
        for cls_name in dir(module):
            cls_obj = getattr(module, cls_name)
            if isinstance(cls_obj, type) and issubclass(cls_obj, TestCase):
                testCases = []
                for test_name in dir(cls_obj):
                    if test_name.startswith(self.testMethodPrefix) and hasattr(getattr(cls_obj, test_name), '__call__'):
                        testCases.append(cls_obj(test_name))
                testSuite = TestSuite(testCases)
                tests.append(testSuite)

        return TestSuite(tests)

defaultTestLoader = TestLoader()


class TestSuite(object):
    '''组合模式，包含TestCase和TestSuite'''

    def __init__(self, tests):
        self.tests = []
        self.addTests(tests)

    def __iter__(self):
        return iter(self.tests)
    
    def addTests(self, tests):
        for test in tests:
            self.tests.append(test)

    def run(self, result):
        for test in self:
            test.run(result)
        return result


class TestCase(object):

    def __init__(self, methodName):
        self._testMethodName = methodName

    def setUp(self):
        pass

    def tearDown(self):
        pass

    def run(self, result):
        result.startTest(self)

        success = False
        testMethod = getattr(self, self._testMethodName)
       
        try:                    
            self.setUp()
        except:
            result.addError(self, sys.exc_info())
        else:
            try:
                testMethod()
            except:
                result.addFailure(self, sys.exc_info())
            else:
                result.addSuccess(self)
                success = True

            try:
                self.tearDown()
            except:
                result.addError(self, sys.exc_info())
        finally:
            result.finishTest(self, success)


class TestResult(object):
    def __init__(self, stream=None, descriptions=None, verbosity=None):
        self.success = 0
        self.failures = 0
        self.errors = 0
        self.testsRun = 0

    #在所有测试开始前运行且只运行一次
    def startTestRun(self):
        self.startTime = time.time()
    
    #在所有测试结束后运行且只运行一次
    def finishTestRun(self):
        self.stopTime = time.time()

    #在每个测试开始前运行
    def startTest(self, test):
        self.testsRun += 1

        print('='*80)
        print(f'run: {test.__class__}.{test._testMethodName}')

    #在每个测试结束后运行
    def finishTest(self, test, success):
        status = 'SUCCESS' if success else 'FAILED'
        print('-'*80)
        print(f'result: {status}')

    def addError(self, test, err):
        self.errors += 1
        print(self._exc_info_to_string(err))

    def addFailure(self, test, err):
        self.failures += 1
        print(self._exc_info_to_string(err))

    def addSuccess(self, test):
        self.success += 1

    def _exc_info_to_string(self, err):
        """Converts a sys.exc_info()-style tuple of values into a string."""
        exctype, value, tb = err
        msgLines = traceback.format_exception(exctype, value, tb)
        return ''.join(msgLines)

    def summary(self):
        timeTaken = self.stopTime - self.startTime
        print('='*80)
        print(f'run {self.testsRun} tests in {timeTaken:.2f} secondes, success:{self.success} fail:{self.failures} error:{self.errors}')


class TestRunner(object):
    def run(self, test):
        result = TestResult()
        result.startTestRun()
        test.run(result)
        result.finishTestRun()

        result.summary()
#-------------
# Test Demo
#-------------
class TestDemo(TestCase):
    def setUp(self):
        print ("do something before test : prepare environment.")

    def tearDown(self):
        print ("do something after test : clean up.")

    def test_add(self):
        assert 1+2 == 3

    def test_minus(self):
        assert 5-3 != 2

    def test_three(self):
        raise Exception("test exception")


if __name__ == '__main__':
    testsuite = defaultTestLoader.loadTestsFromModule(sys.modules[__name__])
    runner = TestRunner()
    result = runner.run(testsuite)
```

## 思考

这样一个框架用来做基本的测试没有什么问题，但是随着需求的变多，在框架上需要不断的进行扩展，显然unittest不满足扩展性，随之而来的就是nose，pytest，robot等的出现，在预置大量fixture的情况下还可以方便的进行扩展，后面的文章我们继续进行分析。
