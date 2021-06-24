pluggy是pytest的插件管理和hook调用的核心，整个pytest也是由多个pluggy插件搭建起来的，为了深入了解pytest的执行过程，我们需要首先来看下pluggy是如何工作的。

用一个[官网](https://pluggy.readthedocs.io/en/latest/)的例子来一步步进行代码分析，深入框架底层：

```python
import pluggy

hookspec = pluggy.HookspecMarker("myproject")
hookimpl = pluggy.HookimplMarker("myproject")

class MySpec(object):
    """A hook specification namespace."""
    @hookspec
    def myhook(self, arg1, arg2):
        """My special little hook that you can customize."""

class Plugin_1(object):
    """A hook implementation namespace."""
    @hookimpl
    def myhook(self, arg1, arg2):
        print("inside Plugin_1.myhook()")
        return arg1 + arg2

class Plugin_2(object):
    """A 2nd hook implementation namespace."""
    @hookimpl
    def myhook(self, arg1, arg2):
        print("inside Plugin_2.myhook()")
        return arg1 - arg2

# create a manager and add the spec
pm = pluggy.PluginManager("myproject")
pm.add_hookspecs(MySpec)
# register plugins
pm.register(Plugin_1())
pm.register(Plugin_2())
# call our `myhook` hook
results = pm.hook.myhook(arg1=1, arg2=2)
print(results)
```

第一步初始化两个装饰器HookspecMarker，HookimplMarker，用于在相应的func上设置xxx_spec/xxx_impl属性并设置opt，后续PluginManager会根据参数project_name用于判断spec，hook并进行加载。

```python
hookspec = pluggy.HookspecMarker("myproject")
hookimpl = pluggy.HookimplMarker("myproject")
```

用函数装饰器的写法如下

```python
def HookspecMarker(name):
    def _spec(hookwrapper=False, optionalhook=False, tryfirst=False, trylast=False):
        def _wrapper(func):
            setattr(func, name+'_spec', dict(
                        hookwrapper=hookwrapper,
                        optionalhook=optionalhook,
                        tryfirst=tryfirst,
                        trylast=trylast,
                    ))
            return func
        return _wrapper
    return _spec

hookspec = HookspecMarker('test')

@hookspec()
def test(a, b):
    print(a+b)

print(getattr(test,'test_spec'))
test(2,3)
```

初始化PluginManager后第一步要做的是添加spec到self.hook，使用add_hookspecs遍历module或者class，当函数被@hookspec装饰后存在xxx_spec属性，添加到self.hook中，hook可以看做是一个dict类型，设置为name -> HookCaller，这里HookCaller内部带有两个队列 _nonwrappers和 _wrappers，为的是保存同一hook的多个实现并在调用时依次调用。

```python
    def add_hookspecs(self, module_or_class):
        for name in dir(module_or_class):
            spec_opts = self.parse_hookspec_opts(module_or_class, name)
            if spec_opts is not None:
                hc = getattr(self.hook, name, None)
                if hc is None:
                    hc = _HookCaller(name, self._hookexec, module_or_class, spec_opts)
                    setattr(self.hook, name, hc)

    def parse_hookspec_opts(self, module_or_class, name):
        method = getattr(module_or_class, name)
        return getattr(method, self.project_name + "_spec", None)

```

第二步添加实现，使用register，这里我们只看核心代码，遍历plugin，当属性被@hookimpl装饰后存在xxx_impl属性，初始化一个HookImpl并将其加入到对应的HookCaller队列中，如果这个spec没有定义的话则会在这里添加一个相应的定义

```python
    def register(self, plugin, name=None):
        for name in dir(plugin):
            hookimpl_opts = self.parse_hookimpl_opts(plugin, name)
            if hookimpl_opts is not None:
                normalize_hookimpl_opts(hookimpl_opts)
                method = getattr(plugin, name)
                hookimpl = HookImpl(plugin, plugin_name, method, hookimpl_opts)
                hook = getattr(self.hook, name, None)
                if hook is None:
                    hook = _HookCaller(name, self._hookexec)
                    setattr(self.hook, name, hook)
                elif hook.has_spec():
                    self._verify_hook(hook, hookimpl)
                    hook._maybe_apply_history(hookimpl)
                hook._add_hookimpl(hookimpl)
                hookcallers.append(hook)
                
    def _add_hookimpl(self, hookimpl):
        """Add an implementation to the callback chain.
        """
        if hookimpl.hookwrapper:
            methods = self._wrappers
        else:
            methods = self._nonwrappers

        if hookimpl.trylast:
            methods.insert(0, hookimpl)
        elif hookimpl.tryfirst:
            methods.append(hookimpl)
        else:
            # find last non-tryfirst method
            i = len(methods) - 1
            while i >= 0 and methods[i].tryfirst:
                i -= 1
            methods.insert(i + 1, hookimpl)
```

细看队列添加方法，如果是trylast，则添加到队列的开始，如果是tryfirst，则添加到队列的最后，默认添加到最后一个非tryfirst之前，可以猜到调用的顺序是倒序调用。

第三步，最终当我们调用一个pm.hook.myhook(args)的时候，实际是通过pm.hook.name找到HookCaller，执行它multicall，实际调用的是caller._multicall这个方法，将hook_impl倒排reversed(hook_impls)并依次执行返回结果

```python
def _multicall(hook_impls, caller_kwargs, firstresult=False):
    """Execute a call into multiple python functions/methods and return the
    result(s).

    ``caller_kwargs`` comes from _HookCaller.__call__().
    """
    __tracebackhide__ = True
    results = []
    excinfo = None
    try:  # run impl and wrapper setup functions in a loop
        teardowns = []
        try:
            for hook_impl in reversed(hook_impls):

                if hook_impl.hookwrapper:
                    try:
                        gen = hook_impl.function(*args)
                        next(gen)  # first yield
                        teardowns.append(gen)
                    except StopIteration:
                        _raise_wrapfail(gen, "did not yield")
                else:
                    res = hook_impl.function(*args)
                    if res is not None:
                        results.append(res)
                        if firstresult:  # halt further impl calls
                            break
        except BaseException:
            excinfo = sys.exc_info()
    finally:
        if firstresult:  # first result hooks return a single value
            outcome = _Result(results[0] if results else None, excinfo)
        else:
            outcome = _Result(results, excinfo)

        # run all wrapper post-yield blocks
        for gen in reversed(teardowns):
            try:
                gen.send(outcome)
                _raise_wrapfail(gen, "has second yield")
            except StopIteration:
                pass

        return outcome.get_result()
```

总的来说，插件框架层次结构如下，调用时候按照nonwrapper, wrapper倒序调用返回结果。

```python
PluginManager
|- hook (_HookRelay)
   |- hookspec1 (_HookCaller)
   |  |- nonwrapper: [hookimpl, hookimpl2, ...]
   |  |- wrapper: [hookimpl, hookimpl2, ...]
   |- hookspec2 (_HookCaller)
   |  |- nonwrapper: [hookimpl, hookimpl2, ...]
   |  |- wrapper: [hookimpl, hookimpl2, ...]
```

以上就是pluggy实现的整个调用链以及基本分析，其余的一些细节和使用请参考官网说明，后续我们继续来分析一下pytest具体的实现思路。