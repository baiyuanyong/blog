让我们用一个简单的例子分析pytest的整个启动执行过程，可以使用`python -m pdb test.py`来观察调用关系，使用s、n进行调试

```python
#file: test.py
def test_01():
    assert 1+1 == 2

if __name__ == '__main__':
    import pytest
    pytest.main(__file__)
```

在pytest.py中可以看到main来自于config.\__init__中

```python
from _pytest.config import main, UsageError, cmdline, hookspec, hookimpl
```

main做了两件事，首先是准备config，然后调用pytest_cmdline_main开始执行

```python
def _prepareconfig(args=None, plugins=None):
    config = get_config(args, plugins)
    pluginmanager = config.pluginmanager
    try:
        if plugins:
            for plugin in plugins:
                if isinstance(plugin, str):
                    pluginmanager.consider_pluginarg(plugin)
                else:
                    pluginmanager.register(plugin)
        return pluginmanager.hook.pytest_cmdline_parse(
            pluginmanager=pluginmanager, args=args
        )
```

get_config首先要建立一个pluggy插件框架，PytestPluginManager重写了PluginManager的方法`parse_hookspec_opts,parse_hookimpl_opts`，使得符合以`pytest_`开头的函数、方法都可以引入而无需加@hookspec装饰。加了@hookspec定义的都是有特殊属性定义的。

Config用来访问配置，进行参数解析，管理插件系统pluginmanager和访问插件hooks，同时也会调用`pytest_addoption`，对参数进行初始化操作。

对每个预定义的plugin调用import_plugin，实际就是调用了`register`引入hookimpl，这样所有的内部预定义插件都可以加入到相应的hookspec队列中

```python
def get_config(args=None, plugins=None):
    # subsequent calls to main will create a fresh instance
    pluginmanager = PytestPluginManager()
    config = Config(
        pluginmanager,
        invocation_params=Config.InvocationParams(
            args=args or (), plugins=plugins, dir=Path().resolve()
        ),
    )

    for spec in default_plugins:
        pluginmanager.import_plugin(spec)
    return config

#加载完成plugin后实际hook以及impl如下
pytest_addhooks
  _wrappers:    []
  _nonwrappers: []
pytest_addoption
  _wrappers:    []
  _nonwrappers: ['mark', 'main', 'runner', 'fixtures', 'helpconfig', 'python', 'terminal', 'debugging', 'skipping', 'pastebin', 'assertion', 'junitxml', 'resultlog', 'doctest', 'cacheprovider', 'setuponly', 'setupplan', 'stepwise', 'warnings', 'logging', 'faulthandler']
pytest_assertion_pass
  _wrappers:    []
  _nonwrappers: []
pytest_assertrepr_compare
  _wrappers:    []
  _nonwrappers: ['assertion']
pytest_cmdline_main
  _wrappers:    []
  _nonwrappers: ['main', 'helpconfig', 'python', 'cacheprovider', 'mark', 'setuponly', 'setupplan']
pytest_cmdline_parse
  _wrappers:    ['helpconfig']
  _nonwrappers: ['pytestconfig']
pytest_cmdline_preparse
  _wrappers:    []
  _nonwrappers: []
pytest_collect_directory
  _wrappers:    []
  _nonwrappers: []
pytest_collect_file
  _wrappers:    []
  _nonwrappers: ['python', 'doctest']
pytest_collection
  _wrappers:    ['warnings']
  _nonwrappers: ['main', 'assertion']
pytest_collection_finish
  _wrappers:    []
  _nonwrappers: []
pytest_collection_modifyitems
  _wrappers:    []
  _nonwrappers: ['mark', 'main']
pytest_collectreport
  _wrappers:    []
  _nonwrappers: []
pytest_collectstart
  _wrappers:    []
  _nonwrappers: []
pytest_configure
  _wrappers:    []
  _nonwrappers: ['logging', 'pastebin', '4423736904', 'mark', 'python', 'terminal', 'debugging', 'skipping', 'tmpdir', 'junitxml', 'resultlog', 'stepwise', 'warnings', 'faulthandler', 'cacheprovider']
pytest_deselected
  _wrappers:    []
  _nonwrappers: []
pytest_doctest_prepare_content
  _wrappers:    []
  _nonwrappers: []
pytest_enter_pdb
  _wrappers:    []
  _nonwrappers: ['faulthandler']
pytest_exception_interact
  _wrappers:    []
  _nonwrappers: ['faulthandler']
pytest_fixture_post_finalizer
  _wrappers:    []
  _nonwrappers: ['setuponly']
pytest_fixture_setup
  _wrappers:    ['setuponly']
  _nonwrappers: ['fixtures', 'setupplan']
pytest_generate_tests
  _wrappers:    []
  _nonwrappers: ['python']
pytest_ignore_collect
  _wrappers:    []
  _nonwrappers: ['main']
pytest_internalerror
  _wrappers:    []
  _nonwrappers: []
pytest_itemcollected
  _wrappers:    []
  _nonwrappers: []
pytest_itemstart
  _wrappers:    []
  _nonwrappers: []
pytest_keyboard_interrupt
  _wrappers:    []
  _nonwrappers: []
pytest_leave_pdb
  _wrappers:    []
  _nonwrappers: []
pytest_load_initial_conftests
  _wrappers:    []
  _nonwrappers: ['pytestconfig']
pytest_make_collect_report
  _wrappers:    []
  _nonwrappers: ['runner']
pytest_make_parametrize_id
  _wrappers:    []
  _nonwrappers: []
pytest_plugin_registered
  _wrappers:    []
  _nonwrappers: []
pytest_pycollect_makeitem
  _wrappers:    ['python']
  _nonwrappers: ['unittest']
pytest_pycollect_makemodule
  _wrappers:    []
  _nonwrappers: ['python']
pytest_pyfunc_call
  _wrappers:    ['skipping']
  _nonwrappers: ['python']
pytest_report_collectionfinish
  _wrappers:    []
  _nonwrappers: []
pytest_report_from_serializable
  _wrappers:    []
  _nonwrappers: ['reports']
pytest_report_header
  _wrappers:    []
  _nonwrappers: ['helpconfig', 'cacheprovider']
pytest_report_teststatus
  _wrappers:    []
  _nonwrappers: ['terminal', 'runner', 'skipping']
pytest_report_to_serializable
  _wrappers:    []
  _nonwrappers: ['reports']
pytest_runtest_call
  _wrappers:    []
  _nonwrappers: ['runner']
pytest_runtest_logfinish
  _wrappers:    []
  _nonwrappers: []
pytest_runtest_logreport
  _wrappers:    []
  _nonwrappers: []
pytest_runtest_logstart
  _wrappers:    []
  _nonwrappers: []
pytest_runtest_makereport
  _wrappers:    ['skipping']
  _nonwrappers: ['runner', 'unittest']
pytest_runtest_protocol
  _wrappers:    ['unittest', 'faulthandler', 'warnings']
  _nonwrappers: ['runner']
pytest_runtest_setup
  _wrappers:    []
  _nonwrappers: ['nose', 'runner', 'assertion', 'skipping']
pytest_runtest_teardown
  _wrappers:    []
  _nonwrappers: ['runner', 'assertion']
pytest_runtestloop
  _wrappers:    []
  _nonwrappers: ['main']
pytest_sessionfinish
  _wrappers:    []
  _nonwrappers: ['runner', 'assertion']
pytest_sessionstart
  _wrappers:    []
  _nonwrappers: ['runner', 'fixtures']
pytest_terminal_summary
  _wrappers:    ['warnings']
  _nonwrappers: ['runner', 'pastebin']
pytest_unconfigure
  _wrappers:    []
  _nonwrappers: ['mark', 'pastebin', 'junitxml', 'resultlog', 'doctest', 'faulthandler']
pytest_warning_captured
  _wrappers:    []
  _nonwrappers: []
```

配置完成后会调用pytest_cmdline_parse这个hook，调用链如下

```python
hook.pytest_cmdline_parse
  config.parse
    hook.pytest_addhooks
    config._preparse
      hook.pytest_load_initial_conftests
    hook.pytest_cmdline_preparse
```

在parse中，会依次调用pytest_addhooks实现，把用户自定义hook添加到系统中。在_preparse中会调用hook.pytest_load_initial_conftests加载conftest。后续调用pytest_cmdline_preparse，此spec暂时没有相应的实现。

所有准备工作都完成后开始主流程的执行，pytest_cmdline_main，这里从main中开始看

```python
hook.pytest_cmdline_main
  main.wrap_session
    Session()
      #注册自己到plugin中
    config._do_configure()
      hook.pytest_configure
      #各个plugin会把自己的一些自定义类注入到pluginmanager中，为后续的hook调用做准备
    hook.pytest_sessionstart
    _main
      hook.pytest_collection
      hook.pytest_runtestloop
    hook.pytest_sessionfinish
```

整体来说，pytest框架使用config作为插件管理者，初始化加载一系列插件，之后会根据内部的一个hook调用顺序依次执行，在此过程中每个实现都针对各自的目的完成一些工作。

由于整个hook执行过程不容易通过pdb进行断点调试，所以pytest整体的理解到此为止，基本框架建立过程已经分析完毕。具体的一些细节可以通过文档自行查看探索

# 参考

[pytest源码](https://github.com/Mering-Gao/pytest/tree/master/pytest%E6%BA%90%E7%A0%81)

[pluggy插件介绍](http://markshao.github.io/2019/10/01/pluggy-guideline/)

