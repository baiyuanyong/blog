# nose2

nose2也是对unittest做了插件化的扩展，虽然名字上是nose2，不过它并不完全兼容nose，而且插件系统也做了很大的改动，使用事件触发，完全不同于nose的插件系统。下面我们着重分析一下它的插件系统运行原理。

nose2中插件管理组件已经改为session，我们看下session是如何加载并调用hooks，看以下三个方法，可以看出第一步是要把实例化的Plugin()加入到plugins队列中，然后根据hooks定义好的方法把当前plugin注册进去。

```python
#session.py
    def loadPlugins(self, modules=None, exclude=None):
        
        log.debug("Loading plugin modules: %s", all_)
        for module in all_:
            self.loadPluginsFromModule(util.module_from_name(module))
        self.hooks.pluginsLoaded(events.PluginsLoadedEvent(self.plugins))

    def loadPluginsFromModule(self, module):
        ....
        for cls in avail:
            log.debug("Plugin is available: %s", cls)
            plugin = cls(session=self)
            if plugin not in self.plugins:
                self.plugins.append(plugin) #添加插件
            for method in self.hooks.preRegistrationMethods:
                if hasattr(plugin, method):
                    self.hooks.register(method, plugin) #注册实例方法

    def registerPlugin(self, plugin):
        
        log.debug("Register active plugin %s", plugin)
        if plugin not in self.plugins:
            self.plugins.append(plugin)
        for method in self.hooks.methods:
            if hasattr(plugin, method):
                log.debug("Register method %s for plugin %s", method, plugin)
                self.hooks.register(method, plugin) #注册实例方法
```

session.hooks对应的结构如下，register做的只是把hook方法对应的实现添加到队列中，这里也用了一个代理类Hook，用来对多个实现进行调用

```python
hooks: class events.PluginInterface
       - hooks{name: class Hook}
          - method_name1, plugins:[x,y,z]
          - method_name2, plugins:[x,y,z]

    def register(self, method, plugin):
        """Register a plugin for a method.

        :param method: A method name
        :param plugin: A plugin instance

        """
        self.hooks.setdefault(method, self.hookClass(method)).append(plugin)
```

