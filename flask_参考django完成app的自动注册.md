#### 1.设置settigns.py中添加需要配置的app

```
INSTALLED_APPS = [
    'apps.system',
    'apps.algorithm',

]
```

#### 2.设置自动收集的配置类

这一块可以借助参考django的 app自动收集，主要有两个类：

- django.apps.config.AppConfig

下面是django的原码

```python
import inspect
import os
from importlib import import_module

from utils.builtins.module_loading import module_has_submodule, import_string

APPS_MODULE_NAME = 'apps'
VIEWS_MODULE_NAME = 'views' #试图模块的名称
EVENT_MODULE_NAME = 'events' #事件模块的名称
DB_MODEL_MODULE_NAME = 'db_model' #数据库模型名称

class AppConfig:

    def __init__(self, app_name, app_module):
        self.name = app_name
        self.module = app_module
        self.apps = None

        # The following attributes could be defined at the class level in a
        # subclass, hence the test-and-set pattern.

        # Last component of the Python path to the application e.g. 'admin'.
        # This value must be unique across a Django projects.
        if not hasattr(self, 'label'):
            self.label = app_name.rpartition(".")[2]
        if not hasattr(self, 'path'):
            self.path = self._path_from_module(app_module)


        self.models_module = None
        self.views_module = None
        self.events_module = None
        self.db_module = None

        self.models = None

    @classmethod
    def create(cls, entry):
        """
        Factory that creates an app config from an entry in INSTALLED_APPS.
        """
        # create() eventually returns app_config_class(app_name, app_module).
        app_config_class = None
        app_config_name = None
        app_name = None
        app_module = None

        # If import_module succeeds, entry points to the app module.
        try:
            app_module = import_module(entry)
        except Exception:
            pass
        else:
            # If app_module has an apps submodule that defines a single
            # AppConfig subclass, use it automatically.
            # To prevent this, an AppConfig subclass can declare a class
            # variable default = False.
            # If the apps module defines more than one AppConfig subclass,
            # the default one can declare default = True.
            if module_has_submodule(app_module, APPS_MODULE_NAME):
                mod_path = '%s.%s' % (entry, APPS_MODULE_NAME)
                mod = import_module(mod_path)
                # Check if there's exactly one AppConfig candidate,
                # excluding those that explicitly define default = False.
                app_configs = [
                    (name, candidate)
                    for name, candidate in inspect.getmembers(mod, inspect.isclass)
                    if (
                            issubclass(candidate, cls) and
                            candidate is not cls and
                            getattr(candidate, 'default', True)
                    )
                ]
                if len(app_configs) == 1:
                    app_config_class = app_configs[0][1]
                    app_config_name = '%s.%s' % (mod_path, app_configs[0][0])
                else:
                    raise RuntimeError("app config class  must only one ")

        if app_config_class is None:
            try:
                app_config_class = import_string(entry)
            except Exception:
                pass
        if app_name is None:
            try:
                app_name = app_config_class.name
            except AttributeError:
                raise RuntimeError(
                    "'%s' must supply a name attribute." % entry
                )
        try:
            app_module = import_module(app_name)
        except ImportError:
            raise RuntimeError(
                "Cannot import '%s'. Check that '%s.%s.name' is correct." % (
                    app_name,
                    app_config_class.__module__,
                    app_config_class.__qualname__,
                )
            )


        return app_config_class(app_name, app_module)


    def import_views(self):
        # Dictionary of models for this app, primarily maintained in the
        # 'all_models' attribute of the Apps this AppConfig is attached to.

        if module_has_submodule(self.module, VIEWS_MODULE_NAME):
            view_module_name = '%s.%s' % (self.name, VIEWS_MODULE_NAME)
            self.views_module = import_module(view_module_name)

    def import_event(self):
        #动态导入所有的事件模块

        if module_has_submodule(self.module, EVENT_MODULE_NAME):
            event_module_name = '%s.%s' % (self.name, EVENT_MODULE_NAME)
            self.events_module = import_module(event_module_name)

    def import_models(self):
        #导入数据模块
        if module_has_submodule(self.module, DB_MODEL_MODULE_NAME):
            db_module_name = '%s.%s' % (self.name, DB_MODEL_MODULE_NAME)
            self.db_module = import_module(db_module_name)


    def _path_from_module(self, module):
        paths = list(getattr(module, '__path__', []))
        if len(paths) != 1:
            filename = getattr(module, '__file__', None)
            if filename is not None:
                paths = [os.path.dirname(filename)]
            else:
                paths = list(set(paths))
        return paths[0]


    def read(self):
        """
        重写该方法可以在系统启动时初始化每个项目
        :return:
        """


```

- django.apps.registry.Apps

django源码

```python
import threading
from collections import defaultdict, Counter

from conf.settings import INSTALLED_APPS
from utils.builtins.config import AppConfig


class Apps:
    def __init__(self,installed_apps=()):

        if installed_apps is None:
            raise RuntimeError("you must supply an installed_apps argument.")

        self.app_configs = {}
        self.apps_ready = self.ready = False
        self._lock = threading.RLock()
        self.loading = False
        self._pending_operations = defaultdict(list)
        if installed_apps is not None:
            self.populate(installed_apps)

    def populate(self, installed_apps=None):
        """
        Load application configurations and models.

        Import each application module and then each model module.

        It is thread-safe and idempotent, but not reentrant.
        """
        if self.ready:
            return

        # populate() might be called by two threads in parallel on servers
        # that create threads before initializing the WSGI callable.
        with self._lock:
            if self.ready:
                return

            # An RLock prevents other threads from entering this section. The
            # compare and set operation below is atomic.
            if self.loading:
                # Prevent reentrant calls to avoid running AppConfig.ready()
                # methods twice.
                raise RuntimeError("populate() isn't reentrant")
            self.loading = True

            # Phase 1: initialize app configs and import app modules.
            for entry in installed_apps:
                if isinstance(entry, AppConfig):
                    app_config = entry
                else:
                    app_config = AppConfig.create(entry)
                if app_config.label in self.app_configs:
                    raise RuntimeError(
                        "Application labels aren't unique, "
                        "duplicates: %s" % app_config.label)

                self.app_configs[app_config.label] = app_config
                app_config.apps = self

            # Check for duplicate app names.
            counts = Counter(
                app_config.name for app_config in self.app_configs.values())
            duplicates = [
                name for name, count in counts.most_common() if count > 1]
            if duplicates:
                raise RuntimeError(
                    "Application names aren't unique, "
                    "duplicates: %s" % ", ".join(duplicates))

            self.apps_ready = True

            # Phase 2: import all blueprint
            for app_config in self.app_configs.values():
                # 将app的model模块导入到-->self.models_module
                app_config.import_views()
                app_config.import_event()
                # app_config.import_models()
            # Phase 3: run ready() methods of app configs.
            for app_config in self.get_app_configs():
                app_config.ready()

            self.ready = True
    def get_app_configs(self):
        """Import applications and return an iterable of app configs."""
        return self.app_configs.values()
apps = Apps(installed_apps=INSTALLED_APPS)
```

通过上述两个类，就可以自动收集app的配置信息。

这里需要注意一点，每个app必须有一个apps的配置类：

```python

from utils.builtins.config import AppConfig


class SystemConfig(AppConfig):
    name = "apps.system"

    def ready(self):
        print("system ready...")

    def get_blueprint(self):
        '''
        获取应用的蓝图配置
        :return:
        '''
        return [
            ("/sys",sys_bp),
        ]

    def get_all_url_rules(self) -> list:
        """
        用于注册所有的类视图
        :return:
        """
        return [
            ('/compounds', CompoundsView.as_view('CompoundsView')),
            # ('/batch_tasks', BatchTaskView.as_view('BatchTaskView')),

        ]
```



#### 3.在main_app中访问apps来控制自动收集的apps

```python
def init_sys(cls):
    """
        初始化系统
        """
    # 1.c初始化flaskapp
    # 1.加载所有模型应用，并初始化所有的模型应用
    from utils.builtins.registry import apps
    all_apps = apps
    configs = all_apps.get_app_configs()

    # print(configs)
    for config in configs:
        # 注册所有的蓝图
        all_bueprints = getattr(config, "get_blueprint")()
        for url_prefix, bp in all_bueprints:
            # print(all_blueprints)
            # 将所有的蓝图配置对象注册到app应用中
            app.register_blueprint(bp, url_prefix=url_prefix)

            # 注册所有类视图
            if hasattr(config, 'get_all_url_rules'):
                all_url_rules = getattr(config, "get_all_url_rules")()
                for rule, func in all_url_rules:
                    app.add_url_rule(rule, view_func=func)
                    logger.info("All applications have been successfully loaded ")
```
