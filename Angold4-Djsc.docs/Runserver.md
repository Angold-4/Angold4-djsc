## Runserver
**不管是在平时调试 还是项目的运行中 使用最多的命令还是 ```./manage.py runserver```<br>**
**第二篇解析选择 runserver 是想从一个宏观的角度看待```django```过程也许会有些不顺畅<br>和前一篇一样有很多疏漏和不完备的地方 
但是我会尽量克服困难 即使是这样 受限于自身的能力因素 可能无法做到面面俱到<br>**
**希望各位多多指教 给我 ```pull request```本人在此提前谢过！**

```python
Block 0
# manage.py
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
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
对比两个程序我们可以看到 有两个主要的不同 一是最主要的是 manage.py 中为runserver设置了setting module 也就是项目中settings.py文件
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
            
"""
"""


        try:
            settings.INSTALLED_APPS
        except ImproperlyConfigured as exc:
            self.settings_exception = exc
        except ImportError as exc:
            self.settings_exception = exc
        if settings.configured:
            # Start the auto-reloading dev server even if the code is broken.
            # The hardcoded condition is a code smell but we can't rely on a
            # flag on the command class because we haven't located it yet.
            if subcommand == 'runserver' and '--noreload' not in self.argv:
                try:
                    autoreload.check_errors(django.setup)()
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

        self.autocomplete()

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
            self.fetch_command(subcommand).run_from_argv(self.argv)

```


