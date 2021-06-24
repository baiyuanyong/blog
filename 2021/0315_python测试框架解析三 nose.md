# nose

之前我们提到的5大测试组件，在nose中均继承自unittest，所以说nose是对unittest的一个扩展，整个框架使用了插件化的思维，框架的核心在于插件体系。接下来我们对插件体系做一个分析。

## 插件体系

首先看下plugin/base.py这个文件，Plugin逻辑比较简单，所有的插件都要实现options和configure，必须要有属性enabled，name，score，其中score属性在有多个实现时候排序用，enabled默认是False，具体含义看下注释即可。

再看下IPluginInterface这个类，这个类相当于一个hook的接口规范，定义了有哪些接口以及相应的属性，属性在导入接口实现也就是插件时会用到，用于代理类的处理。

```python
class IPluginInterface(object):
    """
    IPluginInterface describes the plugin API. Do not subclass or use this
    class directly.
    """
    def addOptions(self, parser, env):
        """Called to allow plugin to register command-line options with the
        parser. DO NOT return a value from this method unless you want to stop
        all other plugins from setting their options.

        .. warning ::

           DEPRECATED -- implement
           :meth:`options <nose.plugins.base.IPluginInterface.options>` instead.
        """
        pass
    add_options = addOptions
    add_options.deprecated = True
```

接下来看下plugin/manager.py这个文件，先要看下PluginManager这个类，用来加载，管理插件，并代理插件调用。Manager的初始化并没有做什么事情，调用addPlugin也是简单的把插件添加到\_plugins队列中，真正触发是通过configure这个方法，通过代理类PluginProxy调用所有插件的configure进行插件初始化，这也是为什么所有的插件都需要实现这个hook。初始化之后调用hook如config.hook.xxx方法的时候会在\_proxies中建立一个相应的代理PluginProxy，此部分代码参考\__getattr__部分，存在直接返回，不存在则建立一个Proxy代理类。

```python
class PluginManager(object):
    proxyClass = PluginProxy

    #重点，每个hook方法会新建一个PluginProxy的代理，代理多个实现的调用
    def __getattr__(self, call):
        try:
            return self._proxies[call]
        except KeyError:
            proxy = self.proxyClass(call, self._plugins)
            self._proxies[call] = proxy
        return proxy

    #对所有的插件进行一个初始化的工作
    def configure(self, options, config):
        """Configure the set of plugins with the given options
        and config instance. After configuration, disabled plugins
        are removed from the plugins list.
        """
        log.debug("Configuring plugins")
        self.config = config
        cfg = PluginProxy('configure', self._plugins)
        cfg(options, config) #对所有插件调用configure
        enabled = [plug for plug in self._plugins if plug.enabled]
        self.plugins = enabled
        self.sort()
        log.debug("Plugins enabled: %s", enabled)
```
让我们继续看代理类的实现逻辑，在pytest中也使用到了代理，原理相似，实现细节不一样。让我们用`PluginProxy('configure', self._plugins)`来观察具体过程。

1. 判断调用的方法是否在接口定义interface中，不在的话抛异常

2. 根据接口的属性返回yield调用，链式调用或者简单调用，赋值给call

3. 从plugin列表中过滤包含接口的插件并保存

4. 在调用proxy()的时候，实际调用的是proxy.chain或者proy.simple，这里忽略generative属性，方便理解。

   查看实际定义，大部分接口都用了简单调用，意味着只返回第一个结果不为None的处理，其余的都会忽略。带有chainable属性的方法没有看明白是什么意思，返回值如何设置，参数如何传递，实际

```python
class PluginProxy(object):
    interface = IPluginInterface
    def __init__(self, call, plugins):
        try:
            self.method = getattr(self.interface, call)
        except AttributeError:
            raise AttributeError("%s is not a valid %s method"
                                 % (call, self.interface.__name__))
        
        # 1. 判断调用的方法是否在接口定义interface中，不在的话抛异常
        # 2. 根据接口的属性chainable返回链式调用或者简单调用，赋值给call
        # 3. 从plugin列表中过滤包含接口的插件并保存
        # 4. 在调用proxy()的时候，实际调用的是proxy.chain或者proxy.simple，
        #    这里忽略generative属性，方便理解
        
        self.call = self.makeCall(call)
        self.plugins = []
        for p in plugins:
            self.addPlugin(p, call)

    def __call__(self, *arg, **kw):
        return self.call(*arg, **kw)

    def addPlugin(self, plugin, call):
        """Add plugin to my list of plugins to call, if it has the attribute
        I'm bound to.
        """
        meth = getattr(plugin, call, None)
        if meth is not None:
            self.plugins.append((plugin, meth))

    def makeCall(self, call):

        meth = self.method
        if getattr(meth, 'generative', False):
            # call all plugins and yield a flattened iterator of their results
            return lambda *arg, **kw: list(self.generate(*arg, **kw))
        elif getattr(meth, 'chainable', False):
            return self.chain
        else:
            # return a value from the first plugin that returns non-None
            return self.simple

    def chain(self, *arg, **kw):
        """Call plugins in a chain, where the result of each plugin call is
        sent to the next plugin as input. The final output result is returned.
        """
        result = None
        # extract the static arguments (if any) from arg so they can
        # be passed to each plugin call in the chain
        static = [a for (static, a)
                  in zip(getattr(self.method, 'static_args', []), arg)
                  if static]
        for p, meth in self.plugins:
            result = meth(*arg, **kw)
            arg = static[:]
            arg.append(result)
        return result

    def generate(self, *arg, **kw):
        """Call all plugins, yielding each item in each non-None result.
        """
        
    def simple(self, *arg, **kw):
        """Call all plugins, returning the first non-None result.
        """
        for p, meth in self.plugins:
            result = meth(*arg, **kw)
            if result is not None:
                return result
```

## 流程分析

分析完插件结构，接下来返回到主流程中看下程序是如何运行起来的。

1. 同unittest一样，也是由TestProgram启动程序，在makeConfig中建立插件系统和配置config，后续流程同unittest中的流程一样，可以参考unittest中TestProgram部分

```python
#core.py
class TestProgram(unittest.TestProgram):
    def __init__(......):
        if config is None:
            config = self.makeConfig(env, plugins)
        if addplugins:
            config.plugins.addPlugins(extraplugins=addplugins)
        self.config = config
        
    def makeConfig(self, env, plugins=None):
        """Load a Config, pre-filled with user config files if any are
        found.
        """
        cfg_files = self.getAllConfigFiles(env)
        if plugins:
            manager = PluginManager(plugins=plugins)
        else:
            manager = DefaultPluginManager()
        return Config(
            env=env, files=cfg_files, plugins=manager)
```

2. 运行parseArgs，首先就是调用config.configure

```python
    def parseArgs(self, argv):
        """Parse argv and env and configure running environment.
        """
        self.config.configure(argv, doc=self.usage())
        self.createTests()
```

config.configure中首先会通过getParser初始化一个argparser，在这里加载所有插件，并添加options，并解析命令行参数。

```python
#config.py
    def _parseArgs(self, argv, cfg_files):
        parser = ConfiguredDefaultsOptionParser(
            self.getParser(), self.configSection, file_error=warn_sometimes)
        return parser.parseArgsAndConfigFiles(argv[1:], cfg_files) #解析命令行参数
    
    def getParser(self, doc=None):
        parser = self.parserClass(doc) #初始化argparser
        parser.add_option(
            "-V","--version", action="store_true",
            dest="version", default=False,
            help="Output nose version and exit")

        self.plugins.loadPlugins() #加载所有插件
        self.pluginOpts(parser) #处理插件参数options

        self.parser = parser
        return parser
    
    def configure(self, argv=None, doc=None):
        options, args = self._parseArgs(argv, cfg_files)
        
        # When listing plugins we don't want to run them
        if not options.showPlugins:
            self.plugins.configure(options, self) #对所有插件根据参数进行配置
            self.plugins.begin()
```

处理完命令行就开始从系统中加载测试用例到suite中

3. 运行runTests，返回结果

```python
def runTests(self):
        """Run Tests. Returns true on success, false on failure, and sets
        self.success to the same value.
        """
        log.debug("runTests called")
        if self.testRunner is None:
            self.testRunner = TextTestRunner(stream=self.config.stream,
                                             verbosity=self.config.verbosity,
                                             config=self.config)
        plug_runner = self.config.plugins.prepareTestRunner(self.testRunner)
        if plug_runner is not None:
            self.testRunner = plug_runner
        result = self.testRunner.run(self.test)
        self.success = result.wasSuccessful()
        if self.exit:
            sys.exit(not self.success)
        return self.success
```

## 总结

对于nose来说，关键是理解插件系统以及如何调用，至于一个hook在什么时候调用这里不做说明，需要大家自己去查看说明。总的插件运行逻辑如下：

```python
1. 建立配置管理Config
   1. 建立插件管理PluginManager
2. 解析参数parseArgs
   1. 进行配置
      1. 初始化argparser
      2. 加载插件
      3. 设置插件options
      4. 配置插件configure
   2. 创建测试用例
      1. 调用testloader加载测试用例
3. 运行测试用例
   1. 调用testrunner运行测试
```

