## Runserver
**不管是在平时调试 还是项目的运行中 使用最多的命令还是 ```./manage.py runserver```<br>**
**第二篇解析选择 runserver 是想从一个宏观的角度看待```django```过程也许会有些不顺畅<br>和前一篇一样有很多疏漏和不完备的地方 
但是我会尽量克服困难 即使是这样 受限于自身的能力因素 可能无法做到面面俱到<br>**
**希望各位多多指教 给我 ```pull request```本人在此提前谢过！**

```python
Block 0
# manage.py
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks.""" import os
import sys


def main():
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project1.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


if __name__ == '__main__':
    main()

"""
看到上面的manage.py的程序 我有一种熟悉之感:
带着这样的感觉,首先得回答一个问题 就是为什么每次初始化的时候 都得调用 ./manage.py runserver 
为什么不调用上一篇我们解析的 django-admin runserver? 
带着这样的疑惑 我试着调用 django-admin runserver 得到如下报错：

raise ImproperlyConfigured(
django.core.exceptions.ImproperlyConfigured: Requested setting DEBUG, but settings are not configured. 
You must either define the environment variable DJANGO_SETTINGS_MODULE or call settings.configure() before accessing settings.
"""

"""对比一下django-admin.py文件的内容 我想我知道了答案："""

# django/bin/django-admin.py
#!/usr/bin/env python
from django.core import management

if __name__ == "__main__":
    management.execute_from_command_line()


"""
对比两个程序我们可以看到 有两个主要的不同:
一是最主要的是 manage.py 中为runserver设置了setting module 也就是项目中settings.py文件
第二 传递给接口 execute_from_command_line() 的时候 manage.py 直接传递了我们在命令行传递的参数:

比如说在执行 ./manage.py runserver的时候 下段点发现此时的sys.argv是：

(Pdb) sys.argv
['manage.py', 'runserver']

"""

```


```python
Block 1
django/core/management/__init__.py func execute_from_command_line()

def execute_from_command_line(argv=None):
    """Run a ManagementUtility."""
    utility = ManagementUtility(argv)
    utility.execute()

"""具体的细节 上一节已经将的很清楚了 操作和django-admin基本相同 所以这里介绍的时候会省略部分细节 如果你是一章一章看过来的 那你应该会很熟悉"""
"""首先创建 ManagementUtility 的实例 接下来调用这个类的execute()方法"""

django/core/management/__init__.py class ManagementUtility method execute()

    def execute(self):
        """
        Given the command-line arguments, figure out which subcommand is being
        run, create a parser appropriate to that command, and run it.
        """
        try:
            subcommand = self.argv[1]
        except IndexError:
            subcommand = 'help'  # Display help if no arguments were given.
        # Preprocess options to extract --settings and --pythonpath.
        # These options could affect the commands that are available, so they
        # must be processed early.
        parser = CommandParser(usage='%(prog)s subcommand [options] [args]', add_help=False, allow_abbrev=False)
        parser.add_argument('--settings')
        parser.add_argument('--pythonpath')
        parser.add_argument('args', nargs='*')  # catch-all
        try:
            options, args = parser.parse_known_args(self.argv[2:])
            handle_default_options(options)
        except CommandError:
            pass  # Ignore any option errors at this point.
     """import pdb; pdb.set_trace()""" 
"""

-> if settings.configured:
(Pdb) settings
<LazySettings "project1.settings">
(Pdb) settings.configured
True

"""
    
    """

    这里的settings和django-admin startproject的时候的settings不一样了 
    虽然都是LazySettings对象 但是如果我们回去看上一篇的startproject的Lazysettings 发现是这样的：
    
    (Pdb) settings
    <LazySettings [Unevaluated]>

    并且这里的settings.configured为True 说明会执行 if settings.configured 之后的语句
    这就不禁让我们产生一个疑问： settings 对象(LazySettings)是什么时候被创建的? 
    首先 在django/core/management/__init__.py 也就是execute()函数的这个文件的开头 我们可以看到settings是被import的:
    from django.conf import settings  所以我们跳到django/conf/__init__.py

    """

        try:
            settings.INSTALLED_APPS
        except ImproperlyConfigured as exc:
            self.settings_exception = exc
        except ImportError as exc:
            self.settings_exception = exc


        if settings.configured:  # True
            # Start the auto-reloading dev server even if the code is broken.
            # The hardcoded condition is a code smell but we can't rely on a
            # flag on the command class because we haven't located it yet.
            if subcommand == 'runserver' and '--noreload' not in self.argv:  # True
                try:
                    autoreload.check_errors(django.setup)()
                """从字面上来看 就是检查错误 这里不多追究"""
                except Exception:
                    # The exception will be raised later in the child process
                    # started by the autoreloader. Pretend it didn't happen by
                    # loading an empty list of applications.
                    apps.all_models = defaultdict(dict)
                    apps.app_configs = {}
                    apps.apps_ready = apps.models_ready = apps.ready = True

                    # Remove options not compatible with the built-in runserver
                    # (e.g. options for the contrib.staticfiles' runserver).
                    # Changes here require manually testing as described in
                    # #27522.
                    _parser = self.fetch_command('runserver').create_parser('django', 'runserver')
                    _options, _args = _parser.parse_known_args(self.argv[2:])
                    for _arg in _args:
                        self.argv.remove(_arg)

            # In all other cases, django.setup() is required to succeed.
            else:
                django.setup()

        self.autocomplete()  """关于autocomplete的内容 详见上一章(StartProject)"""

        if subcommand == 'help':
            if '--commands' in args:
                sys.stdout.write(self.main_help_text(commands_only=True) + '\n')
            elif not options.args:
                sys.stdout.write(self.main_help_text() + '\n')
            else:
                self.fetch_command(options.args[0]).print_help(self.prog_name, options.args[0])
        # Special-cases: We want 'django-admin --version' and
        # 'django-admin --help' to work, for backwards compatibility.
        elif subcommand == 'version' or self.argv[1:] == ['--version']:
            sys.stdout.write(django.get_version() + '\n')
        elif self.argv[1:] in (['--help'], ['-h']):
            sys.stdout.write(self.main_help_text() + '\n')
        else:
            self.fetch_command(subcommand).run_from_argv(self.argv)  """还是会执行这一个命令"""
            """run_from_argv()的内容见 Block 2"""

        """fetch_command(subcommand)"""

    def fetch_command(self, subcommand):
        """
        Try to fetch the given subcommand, printing a message with the
        appropriate command called from the command line (usually
        "django-admin" or "manage.py") if it can't be found.
        """
        # Get commands outside of try block to prevent swallowing exceptions
        commands = get_commands()

        """先看 get_commands(): 跳转到fetch_command的结尾"""

        try:
            app_name = commands[subcommand]
        except KeyError:
            if os.environ.get('DJANGO_SETTINGS_MODULE'):
                # If `subcommand` is missing due to misconfigured settings, the
                # following line will retrigger an ImproperlyConfigured exception
                # (get_commands() swallows the original one) so the user is
                # informed about it.
                settings.INSTALLED_APPS
            else:
                sys.stderr.write("No Django settings specified.\n")
            possible_matches = get_close_matches(subcommand, commands)
            sys.stderr.write('Unknown command: %r' % subcommand)
            if possible_matches:
                sys.stderr.write('. Did you mean %s?' % possible_matches[0])
            sys.stderr.write("\nType '%s help' for usage.\n" % self.prog_name)
            sys.exit(1)

"""

(Pdb) app_name
'django.contrib.staticfiles'
(Pdb) isinstance(app_name, BaseCommand)
False

"""

        if isinstance(app_name, BaseCommand):
            # If the command is already loaded, use it directly.
            klass = app_name
        else:
            klass = load_command_class(app_name, subcommand)

"""

def load_command_class(app_name, name):
    """
    Given a command name and an application name, return the Command
    class instance. Allow all errors raised by the import process
    (ImportError, AttributeError) to propagate.
    """
    module = import_module('%s.management.commands.%s' % (app_name, name))
    return module.Command()


(Pdb) name
'runserver'
(Pdb) module
<module 'django.contrib.staticfiles.management.commands.runserver' from '/Users/Angold4/WorkSpace/.virtualenv/DJSC/lib/python3.8/site-packages/django/contrib/staticfiles/management/commands/runserver.py'>

和上一节的情况类似 我们直接看django这部分的目录:

staticfiles/
    management/commands/
        runserver.py
        findstatic.py
        collecstatic.py

"""

        return klass

"""
随后创建对应指令文件的Command实例：

(Pdb) klass
<django.contrib.staticfiles.management.commands.runserver.Command object at 0x11146b310>
"""

    @functools.lru_cache(maxsize=None)
    def get_commands():
        """
        Return a dictionary mapping command names to their callback applications.

        Look for a management.commands package in django.core, and in each
        installed application -- if a commands package exists, register all
        commands in that package.

        Core commands are always included. If a settings module has been
        specified, also include user-defined commands.

        The dictionary is in the format {command_name: app_name}. Key-value
        pairs from this dictionary can then be used in calls to
        load_command_class(app_name, command_name)

        If a specific version of a command must be loaded (e.g., with the
        startapp command), the instantiated module can be placed in the
        dictionary in place of the application name.

        The dictionary is cached on the first call and reused on subsequent
        calls.
        """
        commands = {name: 'django.core' for name in find_commands(__path__[0])}

"""

(Pdb) command_dir
'/Users/Angold4/WorkSpace/.virtualenv/DJSC/lib/python3.8/site-packages/django/core/management/commands'
(Pdb) __path__[0]
'/Users/Angold4/WorkSpace/.virtualenv/DJSC/lib/python3.8/site-packages/django/core/management'
(Pdb) find_commands(__path__[0])
['check', 'compilemessages', 'createcachetable', 'dbshell', 'diffsettings', 'dumpdata', 'flush', 'inspectdb', 'loaddata', 'makemessages', 'makemigrations', 'migrate', 'runserver', 'sendtestemail', 'shell', 'showmigrations', 'sqlflush', 'sqlmigrate', 'sqlsequencereset', 'squashmigrations', 'startapp', 'startproject', 'test', 'testserver']

"""

        if not settings.configured:
            return commands
    """
    如果settings 还没有 configured 也就是像上一章startproject的情况 那么django就会将__path__[0] 也就是当前正在执行文件所在的绝对路径中的command文件名称全部拿过来
    (find_commands(__path__[0])) 随后新建一个字典 所有的command的名称均为 "django.core" 这样就可以保证之后创建对应的指令类的实例 随后return
    """

        for app_config in reversed(list(apps.get_app_configs())):
            path = os.path.join(app_config.path, 'management')
            commands.update({name: app_config.name for name in find_commands(path)})
        return commands

    """
    但是这里的情况有所不同 因为由前面可知settings.configured == True 所以不会直接return 而是会继续执行接下来的命令
    这里我有一个小疑问 TODO 就是 reversed(list(apps.get_app_configs())) 究竟代表什么？欢迎Pull Request :-)
    随后 在原来commands的字典中update其他的指令 如果有相同的就覆盖(如runserver) 这样就完成了设置后的指令 至于这个有什么用 我们跳回到原来fetch_command的位置
    """

"""
(Pdb) app_config.path
'/Users/Angold4/WorkSpace/.virtualenv/DJSC/lib/python3.8/site-packages/django/contrib/staticfiles'
(Pdb) path
'/Users/Angold4/WorkSpace/.virtualenv/DJSC/lib/python3.8/site-packages/django/contrib/staticfiles/management'
(Pdb) from pprint import pprint as pp
(Pdb) pp(commands)
{'changepassword': 'django.contrib.auth',
 'check': 'django.core',
 'clearsessions': 'django.contrib.sessions',
 'collectstatic': 'django.contrib.staticfiles',
 'compilemessages': 'django.core',
 'createcachetable': 'django.core',
 'createsuperuser': 'django.contrib.auth',
 'dbshell': 'django.core',
 'diffsettings': 'django.core',
 'dumpdata': 'django.core',
 'findstatic': 'django.contrib.staticfiles',
 'flush': 'django.core',
 'inspectdb': 'django.core',
 'loaddata': 'django.core',
 'makemessages': 'django.core',
 'makemigrations': 'django.core',
 'migrate': 'django.core',
 'remove_stale_contenttypes': 'django.contrib.contenttypes',
 'runserver': 'django.contrib.staticfiles',
 'sendtestemail': 'django.core',
 'shell': 'django.core',
 'showmigrations': 'django.core',
 'sqlflush': 'django.core',
 'sqlmigrate': 'django.core',
 'sqlsequencereset': 'django.core',
 'squashmigrations': 'django.core',
 'startapp': 'django.core',
 'startproject': 'django.core',
 'test': 'django.core',
 'testserver': 'django.core'}
 """

```


```python
Block 2
django/contrib/staticfiles/commands/runserver.py

from django.conf import settings
from django.contrib.staticfiles.handlers import StaticFilesHandler
from django.core.management.commands.runserver import (
    Command as RunserverCommand,
)
"""从这个总文件夹的名称也看得出来 这不是主要的代码(contrib) 所以主要的内容还是在django/core/management/commands/runserver.py 中"""


class Command(RunserverCommand):
    help = "Starts a lightweight Web server for development and also serves static files."

    def add_arguments(self, parser):
        super().add_arguments(parser)
        parser.add_argument(
            '--nostatic', action="store_false", dest='use_static_handler',
            help='Tells Django to NOT automatically serve static files at STATIC_URL.',
        )
        parser.add_argument(
            '--insecure', action="store_true", dest='insecure_serving',
            help='Allows serving static files even if DEBUG is False.',
        )

    def get_handler(self, *args, **options):
        """
        Return the static files serving handler wrapping the default handler,
        if static files should be served. Otherwise return the default handler.
        """
        handler = super().get_handler(*args, **options)
        use_static_handler = options['use_static_handler']
        insecure_serving = options['insecure_serving']
        if use_static_handler and (settings.DEBUG or insecure_serving):
            return StaticFilesHandler(handler)
        return handler

"""和前一章类似 """


```

**以下是引用和附录 大部分是主干代码块需要解析但是不重要的文件<br>
为了不影响到整体逻辑的完整性和易读性 所以单独抽出来 这一部分引用的代码分为三种：**
1. **我弄的不是很清楚的 但是从网上可以找到我认为非常好的答案 于是我会贴上来 并且注明来源方便大家查阅(**)
2. **不方便在代码块区域展示的 如图片和树状图**
3. **官方的文档 我也会添加自己的解释**



**Refrence:  [Settings](https://djangodeconstructed.com/2018/05/25/lazy-django-how-django-accesses-project-settings-setup-part-1/)
| [Getattr](https://blog.csdn.net/fjs_cloud/article/details/40683547)**

```python
django/conf/__init__.py class LazySettings


settings = LazySettings()  """创建LazySettings的实例 也就是我们在Block1导入的"""


class LazySettings(LazyObject):
    """
    A lazy proxy for either global Django settings or a custom settings object.
    The user can manually configure settings prior to using them. Otherwise,
    Django uses the settings module pointed to by DJANGO_SETTINGS_MODULE.
    """
    def _setup(self, name=None):
        """
        Load the settings module pointed to by the environment variable. This
        is used the first time settings are needed, if the user hasnt
        configured settings manually.
        """
        settings_module = os.environ.get(ENVIRONMENT_VARIABLE)
        if not settings_module:
            desc = ("setting %s" % name) if name else "settings"
            raise ImproperlyConfigured(
                "Requested %s, but settings are not configured. "
                "You must either define the environment variable %s "
                "or call settings.configure() before accessing settings."
                % (desc, ENVIRONMENT_VARIABLE))

        self._wrapped = Settings(settings_module)

"""

-> self._wrapped = Settings(settings_module)
(Pdb) settings_module
'project1.settings'
(Pdb) Settings(settings_module)
<Settings "project1.settings">

"""

"""父类 LazyObject 的__init__： self._wrapped 默认为 empty"""

class LazyObject:
    """
    A wrapper for another class that can be used to delay instantiation of the
    wrapped class.

    By subclassing, you have the opportunity to intercept and alter the
    instantiation. If you don't need to do that, use SimpleLazyObject.
    """

    # Avoid infinite recursion when tracing __init__ (#19456).
    _wrapped = None

    def __init__(self):
        # Note: if a subclass overrides __init__(), it will likely need to
        # override __copy__() and __deepcopy__() as well.
        self._wrapped = empty


"""
既然 self._wrapped 默认为empty， 这也就意味着如果我们只是直接调用这个Settings(LazySettings) 
那么 Settings._wrapped 默认为 empty 这也就意味着Settings.configured属性默认为False 这个情况和django-admin startproject的时候很像
"""


    @property
    def configured(self):
        """Return True if the settings have already been configured."""
        return self._wrapped is not empty


"""
那么就说明当我们runserver的时候 一定在某个地方调用了最开始的_setup()方法 设置了self._wrapped 为项目的设置对象
那么究竟在哪里调用呢？通过综合网上查的 和自己的搜寻与理解 我找到了答案:

记得在跳转过来的 if settings.configured: 的位置之前 django尝试调用了settings的INSTALLED_APPS属性 当时下段点的值为：

(Pdb) settings.INSTALLED_APPS
['django.contrib.admin', 'django.contrib.auth', 'django.contrib.contenttypes', 'django.contrib.sessions', 'django.contrib.messages', 'django.contrib.staticfiles']

        try:
            settings.INSTALLED_APPS  """问题就在这里：在Python中:当我们调用一个类的实例的属性的时候 首先Python会检查这个对象的类有没有这个属性"""
                                     """如果没有这个属性 那么就会调用这个类的__getattr__方法 随后查询 再如果没有就会报错"""
                                     """所以说 如果我们调用settings的 INSTALLED_APPS属性 本来在 LazySetting 类中是没有这个属性的 所以就会调用 __getattr__ 方法"""
        except ImproperlyConfigured as exc:
            self.settings_exception = exc
        except ImportError as exc:
            self.settings_exception = exc


        if settings.configured:

"""
    def __getattr__(self, name):
        """Return the value of a setting and cache it in self.__dict__."""
        if self._wrapped is empty:
            self._setup(name)
        val = getattr(self._wrapped, name)
        self.__dict__[name] = val
        return val

"""可以看到 在__getattr__方法中 django调用了self._setup()这个函数 从而完成了self._wrapped的设置 从而settings.configured属性就会为True"""
"""之前还有一些更加详细的过程在wsgi这一章我们会详细介绍的"""

```
