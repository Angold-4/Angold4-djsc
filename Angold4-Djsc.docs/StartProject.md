## From django-admin 
**开始创建Django项目的第一个指令 就是在command-line上输入:**
```python
Block 0
django-admin startproject project1

大概过了一秒 会得到一个树状结构 每个部分用于配置django不同的设置：
project1>
    manage.py
 '   project1>
        __init__.py
        asgi.py
        settings.py
        urls.py
        wsgi.py
```
**你有没有想过 这中间发生了什么？**

**所有的一切 都是从django-admin.py开始的
django-admin的作用和熟悉的manage.py作用相似<br> 都是起到一个用户和django-core的一个操纵接口的作用：**

**from django-admin 我们现在开始：**
```python
Block 1
# django/bin/django-admin.py

from django.core import management

if __name__ == "__main__":
    management.execute_from_command_line() 
    '''
    当用户在命令行输入django-admin的时候 系统会执行django-admin.py文件
    这也就意味着会执行 management.execute_from_command_line()函数
    '''
```
```python
# django/core/management/__init__.py func execute_from_command_line()

def execute_from_command_line(argv=None):
    """Run a ManagementUtility."""
    """
    下个断点 看看 utility又是什么
    """
    import pdb; pdb.set_trace()
    utility = ManagementUtility(argv)
    utility.execute()
    
(Djangotest) ➜  Djangotest django-admin startproject project1
(Pdb) utility
<django.core.management.ManagementUtility object at 0x10b3259a0>
"""
可以看出 utility 是一个 ManagementUtility类的实例 
现在我们去看看 ManagementUtility类是什么 为什么需要创建这个实例?
"""
```
```python
Block 2
# django/core/management/__init__.py class ManagementUtility func __init__()

class ManagementUtility:
    """
    Encapsulate the logic of the django-admin and manage.py utilities.
    """
    def __init__(self, argv=None):
    """
    还是下个断点
    """
        import pdb; pdb.set_trace()
        self.argv = argv or sys.argv[:]
        self.prog_name = os.path.basename(self.argv[0])
        if self.prog_name == '__main__.py':
            self.prog_name = 'python -m django'
        self.settings_exception = None


(Djangotest) ➜  Djangotest django-admin startproject project1
-> self.argv = argv or sys.argv[:]
-> self.prog_name = os.path.basename(self.argv[0])
(Pdb) self.argv
['/Users/yy/.virtualenvs/Djangotest/bin/django-admin', 'startproject', 'project1']
"""
可以看出 这里是在向os的命令行试图获取到我们刚才输入的指令 (sys.argv[:])
为了继续回答还有什么别的原因需要创建这个实例 我们继续回到utility.excute()看看这个函数做了什么：
"""
```
****
```python
Block 3
# django/core/management/__init__.py class ManagementUtility method execute()

    def execute(self):
    """
    这个excute是所有Django命令执行的一个入口 所以会有很多与startproject无关的逻辑
    因此函数比较长 为了良好的阅读体验 我尝试在代码中解析 或者引用别处的代码
    """
        """
        Given the command-line arguments, figure out which subcommand is being
        run, create a parser appropriate to that command, and run it.
        """
        try:
            subcommand = self.argv[1] 
            """很明显 这个subcommand是列表的第二个元素 也就是我们输入的 startapp"""
        except IndexError:
            subcommand = 'help'  # Display help if no arguments were given.
            """如果我们没有给出指令 比如说只是输入了 django-admin 那么subcommand就默认等于help 这也符合我们的逻辑"""

        # Preprocess options to extract --settings and --pythonpath.
        # These options could affect the commands that are available, so they
        # must be processed early.
        parser = CommandParser(usage='%(prog)s subcommand [options] [args]', add_help=False, allow_abbrev=False)
        """创建一个CommandParser的类的实例 翻译过来是指令语法分析程序："""
        """继承自 ArgumentParser 但是这个是Python库中的程序 详情可以参考该程序块下方我查找的解释"""
        parser.add_argument('--settings')
        parser.add_argument('--pythonpath')
        parser.add_argument('args', nargs='*')  # catch-all
        try:
            options, args = parser.parse_known_args(self.argv[2:])
            handle_default_options(options)
        except CommandError:
            pass  # Ignore any option errors at this point.
        """为了不影响主要程序的解析 我这里整理出来了一个方便理解的意思："""
        """1.首先 add_argument的作用是增加一个可选参数 类似于class中的初始化可选参数"""
        """2.其中第一项表示参数名称 其中前面带"--"的表示可选参数 什么都不带的表示必选参数"""
        """3.随后的arg="*"表示这个参数可以有任意个变量(引用中有详细介绍)"""
        """这个parser的作用是什么 我们通过结果来看 在options, args=parser.parse_known_args(self.argv[2:]处下断点)"""
        """
        (Pdb) options
        Namespace(args=['project1'], pythonpath=None, settings=None)
        (Pdb) args
        self = <django.core.management.ManagementUtility object at 0x1069189a0>
        
        可以看到 由于这个excute函数是全部命令的入口 也就是不管你敲什么命令 都会先执行这个函数
        所以这里很多的逻辑会检查一些其他的配置 比如说初始化参数(parser) 检查参数(见下方)
        """
        try:
            settings.INSTALLED_APPS
        except ImproperlyConfigured as exc:
            self.settings_exception = exc
        except ImportError as exc:
            self.settings_exception = exc
        """如果是startproject这类指令 setting.comfigured为False 也就是这个if 不会执行"""
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
            """下面的django.setup() 和 autocomplete是两个django中比较重要的函数 详见文末的解释："""
                django.setup()

        self.autocomplete()

        if subcommand == 'help':
        """如果什么都没填或者填的是help 那么就会执行这个if块 打印出不同的帮助信息"""
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
        """很显然 startproject指令是最后一条"""
        else:
            self.fetch_command(subcommand).run_from_argv(self.argv)
        """接下来我们来看这个fetch_command函数 见Block4 之后再回来继续分析"""
        
        """
        如果分析完fetch_command 就知道此时return了一个klass object
        这个object 是我们创建的startproject.Command类的实例 拥有对应指令的不同处理逻辑 
        下面我们就可以通过run_from_argv()来正式生成我们的目录了:
        run_from_argv()的分析 见Block5
        
        """

```
****
```python
Block 4
# django/core/management/__init__.py class ManagementUtility method fetch_command()
    def fetch_command(self, subcommand):
        """
        Try to fetch the given subcommand, printing a message with the
        appropriate command called from the command line (usually
        "django-admin" or "manage.py") if it can't be found.
        """
        # Get commands outside of try block to prevent swallowing exceptions
        commands = get_commands()
        """
        get_command()函数的逻辑是获取所有的django指令列表：
        
        def get_command()
        commands = {name: 'django.core' for name in find_commands(__path__[0])}
    
        if not settings.configured:
            return commands
    
        for app_config in reversed(list(apps.get_app_configs())):
            path = os.path.join(app_config.path, 'management')
            commands.update({name: app_config.name for name in find_commands(path)})
    
        return commands
        
        我们在最开头下个断点：
        """
        
(Pdb) find_commands(__path__[0])
['check', 'compilemessages', 'createcachetable', 'dbshell', 'diffsettings', 'dumpdata', 'flush', 'inspectdb', 'loaddata', 'makemessages', 'makemigrations', 'migrate', 'runserver', 'sendtestemail', 'shell', 'showmigrations', 'sqlflush', 'sqlmigrate', 'sqlsequencereset', 'squashmigrations', 'startapp', 'startproject', 'test', 'testserver']
(Pdb) cc = commands
(Pdb) from pprint import pprint as pp
(Pdb) pp(cc)
{'check': 'django.core',
 'compilemessages': 'django.core',
 'createcachetable': 'django.core',
 'dbshell': 'django.core',
 'diffsettings': 'django.core',
 'dumpdata': 'django.core',
 'flush': 'django.core',
 'inspectdb': 'django.core',
 'loaddata': 'django.core',
 'makemessages': 'django.core',
 'makemigrations': 'django.core',
 'migrate': 'django.core',
 'runserver': 'django.core',
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
        可以看到 command是一个字典 只不过这个字典的索引是指令的名称 而值都是django.core 随后我们会发现这样写的意义
        """
        try:
            app_name = commands[subcommand]
            """这样就可以获取到对应的app_name 也就是django.core"""
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
        if isinstance(app_name, BaseCommand):
            # If the command is already loaded, use it directly.
            klass = app_name
        """如果这个app_name是BaseCommand类的实例的话 那么直接不做处理返回klass"""
        else:
        """我们这里的是这样的情况 那么我们就会创建一个BaseCommand类的实例"""
            klass = load_command_class(app_name, subcommand)
        return klass
        """在这里下一个断点 看看klass是什么"""
(Pdb) klass
<django.core.management.commands.startproject.Command object at 0x110080fd0>
        """下面我们看看load command这个函数 看看是怎样创建的这个类的实例："""
        
    def load_command_class(app_name, name):
    """
    Given a command name and an application name, return the Command
    class instance. Allow all errors raised by the import process
    (ImportError, AttributeError) to propagate.
    """
    module = import_module('%s.management.commands.%s' % (app_name, name))
    return module.Command()
    """
    可以看到 这里module实际上是 django.core.management.commands.startproject.py
    这也解释了值是django.core的作用 对应着将要从哪个django部分导入这个module
    而进入到django.core.management.commands这个文件夹 我们可以验证这一点：
    commands>
        check.py
        compilemessages.py
        createcachetable.py
        dbshell.py
        diffsettings.py
        dumpdata.py
        flush.py
        inspectdb.py
        loaddata.py
        makemessages.py
        makemigrations.py
        migrate.py
        runserver.py
        sendtestemail.py
        shell.py
        showmigrations.py
        sqlflush.py
        sqlmigrate.py
        sqlsequencereset.py
        squashmigrations.py
        startapp.py
        startproject.py
        test.py
        testserver.py
    """
    """
    下面进入startproject.py 看看这个类 Command
    """
    class Command(TemplateCommand):
    help = (
        "Creates a Django project directory structure for the given project "
        "name in the current directory or optionally in the given directory."
    )
    missing_args_message = "You must provide a project name."

    def handle(self, **options):
        project_name = options.pop('name')
        target = options.pop('directory')

        # Create a random SECRET_KEY to put it in the main settings.
        options['secret_key'] = get_random_secret_key()

        super().handle('project', project_name, target, **options)
    
    """
    讲到这里 类Utility的任务就完成了 一开始通过ManagementUtility(argv)创建这个类之后 
    我们获取到了用户输入的command是那一部分的哪个指令 并进行校验
    最后我们在这个对应指令的文件下创建了对应指令的Command类的实例 下面就可以继续进行操作了
    回见 block 3
    """
    
```
****
```python
Block 5
# django/core/management/base.py class BaseCommand method run_from_argv()
    
    """从这个函数的名称也看得出来 run_from_argv 而argv是我们之前输入的startproject"""
    def run_from_argv(self, argv):
        """
        Set up any environment changes requested (e.g., Python path
        and Django settings), then run this command. If the
        command raises a ``CommandError``, intercept it and print it sensibly
        to stderr. If the ``--traceback`` option is present or the raised
        ``Exception`` is not ``CommandError``, raise it.
        """
        
        self._called_from_command_line = True
        parser = self.create_parser(argv[0], argv[1])
        options = parser.parse_args(argv[2:])
        cmd_options = vars(options)
        """vars函数是将 options对象的每个属性单独提取出来 整合成一个字典"""
        # Move positional args out of options to mimic legacy optparse
        args = cmd_options.pop('args', ())
    
        """下断点分别看看options 和 parser 以及 cmd_options 是什么:"""
        """
(Pdb) options
Namespace(directory=None, extensions=['py'], files=[], force_color=False, name='project1', no_color=False, pythonpath=None, settings=None, template=None, traceback=False, verbosity=1)
(Pdb) parser
CommandParser(prog='django-admin startproject', usage=None, description='Creates a Django project directory structure for the given project name in the current directory or optionally in the given directory.', formatter_class=<class 'django.core.management.base.DjangoHelpFormatter'>, conflict_handler='error', add_help=True)
        """
        """
(Pdb) cmd_options
{'verbosity': 1, 'settings': None, 'pythonpath': None, 'traceback': False, 'no_color': False, 'force_color': False, 'name': 'project1', 'directory': None, 'template': None, 'extensions': ['py'], 'files': []}
        """
        """再来看看args是什么 这里设置好了当前执行django-admin的目录 以及指令名称"""
        """
(Pdb) args
self = <django.core.management.commands.startproject.Command object at 0x104d63df0>
argv = ['/Users/yy/.virtualenvs/Djangotest/bin/django-admin', 'startproject', 'project1']
        """
        
        handle_default_options(options)
        try:
            self.execute(*args, **cmd_options)
        """之后都是execute出错的逻辑 我们现在来看看excute执行的逻辑 详见Block 6"""
        except Exception as e:
            if options.traceback or not isinstance(e, CommandError):
                raise

            # SystemCheckError takes care of its own formatting.
            if isinstance(e, SystemCheckError):
                self.stderr.write(str(e), lambda x: x)
            else:
                self.stderr.write('%s: %s' % (e.__class__.__name__, e))
            sys.exit(1)
        finally:
            try:
                connections.close_all()
            except ImproperlyConfigured:
                # Ignore if connections aren't setup at this point (e.g. no
                # configured settings).
                pass


```
****
```python
Block 6
# django/core/management/base.py class BaseCommand method execute()

    def execute(self, *args, **options):
    
        if options['force_color'] and options['no_color']:
            raise CommandError("The --no-color and --force-color options can't be used together.")
        if options['force_color']:
            self.style = color_style(force_color=True)
        elif options['no_color']:
            self.style = no_style()
            self.stderr.style_func = None
        if options.get('stdout'):
            self.stdout = OutputWrapper(options['stdout'])
        if options.get('stderr'):
            self.stderr = OutputWrapper(options['stderr'])

        if self.requires_system_checks and not options['skip_checks']:
            self.check()
        if self.requires_migrations_checks:
            self.check_migrations()
  
        """到这一步之前 都是在做检查以及样式颜色的配置 我们可以不多考虑(此处参考the5fire老师的逻辑)"""
        """直接看self.handle函数 可以看到 这两个参数分别是args列表 和 cmd_options字典"""
        output = self.handle(*args, **options)
        """因为self是一个startproject.Command()实例 所以我们回过头看这个方法 详见本Block excute()本函数的末尾"""
        if output:
            if self.output_transaction:
                connection = connections[options.get('database', DEFAULT_DB_ALIAS)]
                output = '%s\n%s\n%s' % (
                    self.style.SQL_KEYWORD(connection.ops.start_transaction_sql()),
                    output,
                    self.style.SQL_KEYWORD(connection.ops.end_transaction_sql()),
                )
            self.stdout.write(output)
        return output
        
        
# django/core/management/commands/startproject.py

class Command(TemplateCommand):
    help = (
        "Creates a Django project directory structure for the given project "
        "name in the current directory or optionally in the given directory."
    )
    missing_args_message = "You must provide a project name."
    
    """可以看到这里有handle方法 但是是继承了父类的方法 所以我们去父类TemplateCommand寻找这个方法："""
    """因为代码较长 handle方法 详见Block 7"""
    def handle(self, **options):
        project_name = options.pop('name')
        target = options.pop('directory')
        
        """
(Pdb) options
{'verbosity': 1, 'settings': None, 'pythonpath': None, 'traceback': False, 'no_color': False, 'force_color': False, 'name': 'project1', 'directory': None, 'template': None, 'extensions': ['py'], 'files': []}
        """

        # Create a random SECRET_KEY to put it in the main settings.
        options['secret_key'] = get_random_secret_key()
    """可以看到 在每个project中的settings.py文件中都有的独特的secret_key就是在这是生成的"""
        super().handle('project', project_name, target, **options)


```
![image](http://note.youdao.com/yws/res/22880/WEBRESOURCE5d1ca2ec44ca4a1d4648cd6735ef7176)
****
```python
Block 7
#django/core/management/templates.py class TemplateCommand method handle


    """这一部分的代码 是整个django-admin startproject 或者 startapp中最重要的逻辑"""
    """可以说 如果前面都不看 只看这一部分的逻辑 那么也可以大致理解总体的逻辑 只剩一些小细节"""
    """
    回顾之前 创建了utility的instance来获取用户输入的具体指令 
    经过校验以及配置之后 并创建具体的指令的类的Command实例 将配置项以option作为字典的方式 
    一步一步 传递给最后执行的handle函数 也就是接下来的函数 
    前面的一步一步配置和传递打下的基础之后 我们终于可以开始最后的执行了:
    """
    def handle(self, app_or_project, name, target=None, **options):
        self.app_or_project = app_or_project
        self.a_or_an = 'an' if app_or_project == 'app' else 'a'
        self.paths_to_remove = []
        self.verbosity = options['verbosity']

        self.validate_name(name)

        # if some directory is given, make sure it's nicely expanded
        """如果target是None的话 就像我们startproject的情况 target对应directory=None"""
        if target is None:
            top_dir = os.path.join(os.getcwd(), name)
        """top_dir表示创建app/project的根目录"""
        """对python.os模块熟知的同学肯定会理解""" 
        """这个os.getcwd()会返回当前的执行django-admin的目录名 而os.path.join的作用是路径拼接"""
        """详情见本文最后的附录部分搜索os"""
        """
        这里的top_dir 下个断点看看具体的值:
(Pdb) top_dir
'/Users/yy/Desktop/Project/Django/Djangotest/project1/project1/project1'
        """
            try:
                os.makedirs(top_dir)
        """os.mkdirs是创建当前的路径 如果已经存在就不会创建 不存在的情况是我们指定路径的情况 也就是target!=None的情况"""
            except FileExistsError:
                raise CommandError("'%s' already exists" % top_dir)
            except OSError as e:
                raise CommandError(e)
        else:
            if app_or_project == 'app':
                self.validate_name(os.path.basename(target), 'directory')
            top_dir = os.path.abspath(os.path.expanduser(target))
            if not os.path.exists(top_dir):
                raise CommandError("Destination directory '%s' does not "
                                   "exist, please create it first." % top_dir)

        extensions = tuple(handle_extensions(options['extensions']))
        """extensions 表示是文件的后缀 这里options['extensions']=['py']表示python文件"""
        extra_files = []
        for file in options['files']:
            extra_files.extend(map(lambda x: x.strip(), file.split(',')))
        """在直接执行django-admin startproject 的逻辑中 file为空 可以查看在Block6末尾的options获取详情"""
        if self.verbosity >= 2:
        """options[verbosity]是1 所以不会执行这个语句 可以在文末查询verbosity查看verbosity的作用"""
            self.stdout.write("Rendering %s template files with "
                              "extensions: %s\n" %
                              (app_or_project, ', '.join(extensions)))
            self.stdout.write("Rendering %s template files with "
                              "filenames: %s\n" %
                              (app_or_project, ', '.join(extra_files)))

        base_name = '%s_name' % app_or_project
        base_subdir = '%s_template' % app_or_project
        base_directory = '%s_directory' % app_or_project
        camel_case_name = 'camel_case_%s_name' % app_or_project
        """name就是传过来的project_name 就是project 这样做(join)的目的是"_"最好不用作文件名或者是隐藏变量的逻辑(欢迎纠错)"""
        """name_title()表示的是将字符串name的首字母大写 随后返回"""
        camel_case_value = ''.join(x for x in name.title() if x != '_')
        """
        统一下断点输出一下几个变量的值：
        
(Pdb) base_name
'project_name'
(Pdb) base_directory
'project_directory'
(Pdb) base_subdir
'project_template'
(Pdb) camel_case_name
'camel_case_project_name'
(Pdb) camel_case_value
'Project1'

        """

        context = Context({
            **options,
            base_name: name,
            base_directory: top_dir,
            camel_case_name: camel_case_value,
            'docs_version': get_docs_version(),
            'django_version': django.__version__,
        }, autoescape=False)
        
        """
        同样的 下断点看看context的值
    
(Pdb) pp(context)
[{'True': True, 'False': False, 'None': None}, {'verbosity': 1, 'settings': None, 'pythonpath': None, 'traceback': False, 'no_color': False, 'force_color': False, 'template': None, 'extensions': ['py'], 'files': [], 'secret_key': 'w6i$9365vq@9n2zrf3pc=0d7&)71&f8o)dq!$p*&g^ow#q87pq', 'project_name': 'project1', 'project_directory': '/Users/yy/Desktop/Project/Django/Djangotest/project1/project1/project1', 'camel_case_project_name': 'Project1', 'docs_version': '3.0', 'django_version': '3.0.5'}]

        """

        # Setup a stub settings environment for template rendering
        if not settings.configured:
            settings.configure()
            django.setup()
        """代码过长 会导致逻辑看起来复杂 我将django.setup()和settings.configure()两个函数的解析逻辑放在文末""" 
        """可以搜索setup()和settings.configure()查看"""
        
```
```python
        """接下来就是渲染模版的逻辑"""
        template_dir = self.handle_template(options['template'],
                                            base_subdir)
        """
        其实handle_template实际上执行的代码就两行：
        """
        def handle_template(self, template, subdir):
            """
            Determine where the app or project templates are.
            Use django.__path__[0] as the default because the Django install
            directory isn't known.
            """
            """从上面的简介中可以看出 这个函数只做了一件事——在django源码中找到对应模版的位置"""
            if template is None:
                return os.path.join(django.__path__[0], 'conf', subdir)
            """os.path.join之前也提过 拼接地址:django.__path__[0](django的位置)/conf/project_template"""
            """下一个断点看看 template_dir的位置: 以及prefix_length的值"""
            """
            
(Pdb) template_dir
'/Users/yy/.virtualenvs/Djangotest/lib/python3.8/site-packages/django/conf/project_template'

(Pdb) prefix_length
91

            """

        """
        """
        prefix_length = len(template_dir) + 1

        for root, dirs, files in os.walk(template_dir):
        """os.walk(path)表示遍历path路径下的所有文件 详见文末的os块"""

            path_rest = root[prefix_length:]
            relative_dir = path_rest.replace(base_name, name)
            if relative_dir:
                target_dir = os.path.join(top_dir, relative_dir)
                os.makedirs(target_dir, exist_ok=True)

            for dirname in dirs[:]:
                if dirname.startswith('.') or dirname == '__pycache__':
                    dirs.remove(dirname)

            for filename in files:
                if filename.endswith(('.pyo', '.pyc', '.py.class')):
                    # Ignore some files as they cause various breakages.
                    continue
                old_path = os.path.join(root, filename)"""这里是遍历文件的逻辑位置(也就是manage.py-tpl)"""
                new_path = os.path.join(
                    top_dir, relative_dir, filename.replace(base_name, name)
                )
                for old_suffix, new_suffix in self.rewrite_template_suffixes:
                """
                这里的逻辑是：因为模版文件都有一个后缀 .py-tpl 
                我们在创建目录的时候需要将这个后缀换成project中的.py 也就是new_suffix
                
(Pdb) self.rewrite_template_suffixes
(('.py-tpl', '.py'),)

                """
                    if new_path.endswith(old_suffix):
                        new_path = new_path[:-len(old_suffix)] + new_suffix
                        break  # Only rewrite once

                if os.path.exists(new_path):
                    raise CommandError(
                        "%s already exists. Overlaying %s %s into an existing "
                        "directory won't replace conflicting files." % (
                            new_path, self.a_or_an, app_or_project,
                        )
                    )

                # Only render the Python files, as we don't want to
                # accidentally render Django templates files
                """如果new_path：manage.py-tpl 是以.py结尾 那么就open这个文件"""
                if new_path.endswith(extensions) or filename in extra_files:
                    with open(old_path, encoding='utf-8') as template_file:
                        content = template_file.read()
                        """read()会获取到这个文件所有的字符串"""
            """
            
(Pdb) print(content)
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys


def main():
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', '{{ project_name }}.settings')
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

(Pdb)

            """
                    template = Engine().from_string(content)
                    content = template.render(context)
                    with open(new_path, 'w', encoding='utf-8') as new_file:
                        new_file.write(content)
            """
            Engine()是Django自带的模版逻辑 
            content指将我们的的模版文件中的变量{{}}名称改写为context字典中所有的参数对应的值
            这一步其实和view层的逻辑十分相似 做一个模版 剩下的只需要传一个context字典按照变量名填值即可
            现在content我们创建的文件的内容了
            接下来就在new_path也就是创建文件的路径下 创建一个新的文件(new_path自带文件名所以无需取名)
            随后将content渲染好的内容write进去
            这样就创建好了一个新的文件manage.py
            """
            """
            
(Pdb) print(content)
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
            """
            如果有同学疑惑: 这样只创建好了主目录下files的逻辑 那么子目录下的files怎么办？
            可以看到 上面所有的逻辑都是在一个for循环下的：
            for root, dirs, files in os.walk(template_dir):
            这个循环会一步步遍历所有的子文件和子文件夹 然后渲染模版 
            比如说渲染完manage.py之后 再来执行一次循环 遍历子目录下的文件：
            
(Pdb) top_dir
'/Users/yy/Desktop/Project/Django/Djangotest/project1/project1/project1'
(Pdb) files
['__init__.py-tpl', 'urls.py-tpl', 'asgi.py-tpl', 'wsgi.py-tpl', 'settings.py-tpl']
(Pdb) name
'project1'
(Pdb) relative_dir
'project1'
            
            也可以看到 如果是创建自文件夹 比如说这里的Project1/Project1 
            那么因为relative_dir不是空了 那么就会创建名为Project1的子文件夹
            在这个下面创建文件文件 随后就会把这个relative文件夹拼接起来
            """
                else:
                    shutil.copyfile(old_path, new_path)

                if self.verbosity >= 2:
                    self.stdout.write("Creating %s\n" % new_path)
                try:
                    shutil.copymode(old_path, new_path)
                    self.make_writeable(new_path)
                except OSError:
                    self.stderr.write(
                        "Notice: Couldn't set permission bits on %s. You're "
                        "probably using an uncommon filesystem setup. No "
                        "problem." % new_path, self.style.NOTICE)

        if self.paths_to_remove:
            if self.verbosity >= 2:
                self.stdout.write("Cleaning up temporary files.\n")
            for path_to_remove in self.paths_to_remove:
                if os.path.isfile(path_to_remove):
                    os.remove(path_to_remove)
                else:
                    shutil.rmtree(path_to_remove)

```
****
```python
Block 8

输出一下context:
{'extensions': ['py'],
 'files': [],
 'force_color': False,
 'no_color': False,
 'pythonpath': None,
 'secret_key': 'q!@dy$ddor4&^4p0bq54)1$dvf#-3(x)iw)kp=!-0m&dng0pp^',
 'settings': None,
 'template': None,
 'traceback': False,
 'verbosity': 1
 'base_name:' name,
 'base_directory:' top_dir,
 'camel_case_name:' camel_case_value,
 'docs_version':'get_docs_version(),
 'django_version': django.__version__,}

```
*****
**以下是引用和附录 大部分是主干代码块需要解析但是不重要的文件<br>
为了不影响到整体逻辑的完整性和易读性 所以单独抽出来 这一部分引用的代码分为三种：**
1. **我弄的不是很清楚的 但是从网上可以找到我认为非常好的答案 于是我会贴上来 并且注明来源方便大家查阅(**)
2. **不方便在代码块区域展示的 如图片和树状图**
3. **官方的文档 我也会添加自己的解释**

*****
* **Python argparse** **[python docs](https://docs.python.org/3/library/argparse.html) | [stackoverflow](https://stackoverflow.com/questions/20165843/argparse-how-to-handle-variable-number-of-arguments-nargs)**
![image](http://note.youdao.com/yws/res/22540/WEBRESOURCE75dc15358ddd777f36e886eca7b6e1da)
* **settings.configure() [django docs](https://docs.djangoproject.com/en/3.0/topics/settings/)<br>**

```python
# django/conf/__init__.py class LazySettings method configure()

    def configure(self, default_settings=global_settings, **options):
    """default_settings=global_settings 这条比较关键""" 
    """大家可以看看django/conf/global_settings.py 这是一个有639行的文件 包含了各种的settings(实际默认创建的settings只有120行左右)"""
        """
        Called to manually configure the settings. The 'default_settings'
        parameter sets where to retrieve any unspecified values from (its
        argument must support attribute access (__getattr__)).
        """
        if self._wrapped is not empty:
            raise RuntimeError('Settings already configured.')
        """可以看到 最后返回的是_wrapped 而_wrapped = holder 
        而holder是UserSettingsHoder的一个实例"""
        holder = UserSettingsHolder(default_settings)
    
        for name, value in options.items():
            if not name.isupper():
                raise TypeError('Setting %r must be uppercase.' % name)
            setattr(holder, name, value)
        self._wrapped = holder

```
* **django.setup() [django docs](https://docs.djangoproject.com/en/3.0/ref/applications/#django.setup)<br>**
这个函数的位置在```django/__init__.py```是一个比较重要的函数 下面我们来解析(在之后可能会反复引用)：

```python
# django.__init__.py func setup()

def setup(set_prefix=True):
    """
    Configure the settings (this happens as a side effect of accessing the
    first setting), configure logging and populate the app registry.
    Set the thread-local urlresolvers script prefix if `set_prefix` is True.
    """
    from django.apps import apps
    from django.conf import settings
    from django.urls import set_script_prefix
    from django.utils.log import configure_logging

    configure_logging(settings.LOGGING_CONFIG, settings.LOGGING)
    if set_prefix:
        set_script_prefix(
            '/' if settings.FORCE_SCRIPT_NAME is None else settings.FORCE_SCRIPT_NAME
        )
    apps.populate(settings.INSTALLED_APPS)

'''
这个函数的作用主要是配置设置
其中有一个set_script_prefix函数的逻辑到了url的逻辑 
这里挖一个小坑等到解析url的时候来填 TODO
'''
'''我们主要来看之后执行的函数 apps.populate(settings.INSTALLED_APPS)'''
```
```python
# django/apps/registry.py class Apps method populate

    def populate(self, installed_apps=None):
        """
        Load application configurations and models.

        Import each application module and then each model module.

        It is thread-safe and idempotent, but not reentrant.
        """
        if self.ready:
            return
        """如果ready = true了那么就退出循环 setup完毕"""

        # populate() might be called by two threads in parallel on servers
        # that create threads before initializing the WSGI callable.
        with self._lock: """加了一个锁 防止不同的进程相互干扰"""
            if self.ready:
                return
            """如果ready = true了那么就退出循环 setup完毕"""

            # An RLock prevents other threads from entering this section. The
            # compare and set operation below is atomic.
            if self.loading:
                # Prevent reentrant calls to avoid running AppConfig.ready()
                # methods twice.
                raise RuntimeError("populate() isn't reentrant")
            self.loading = True

            # Phase 1: initialize app configs and import app modules.
            """第一步 检测app的有关设置(名字等等)"""
            for entry in installed_apps:
                if isinstance(entry, AppConfig):
                    app_config = entry
                else:
                    app_config = AppConfig.create(entry)
                    """检查app名字是否重名"""
                if app_config.label in self.app_configs:
                    raise ImproperlyConfigured(
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
                raise ImproperlyConfigured(
                    "Application names aren't unique, "
                    "duplicates: %s" % ", ".join(duplicates))

            self.apps_ready = True

            # Phase 2: import models modules.
            """第二步 import app需要的模块"""
            for app_config in self.app_configs.values():
                app_config.import_models()

            self.clear_cache()

            self.models_ready = True

            # Phase 3: run ready() methods of app configs.
            """第三步 检查执行是否成功逻辑"""
            for app_config in self.get_app_configs():
                app_config.ready()

            self.ready = True
            self.ready_event.set()
    """
    从这个函数的整个逻辑和注释可以很清楚的看到 这一步其实就是执行app的初始化设置 
    如果全部都设置好了那么就执行完毕 setup()也就执行完毕
    这里的逻辑不像是startproject的逻辑 更像是runserver前的check逻辑 这里我猜测 原因有二：
    第一个原因是如果是startapp的话 那么根本不会执行这些逻辑 因为如果我在setup()之前的代码块下断点 发现根本不会执行setup()的逻辑：
    
    -> if settings.configured:
    (Pdb) settings.configured
    False
    
    第二个原因是 我们的startproject并没有 INSTALLED_APP 所以执行setup()的用处微乎其微
    """
```
* **autocomplete<br>**
**这个函数是自动补全函数 触发条件是tab**
```python
"""
接下来看下一个函数 autocomplete() 这个函数也是Linux系统中command应用自动补全的通用逻辑
"""
# django/core/management/__init__.py class ManagementUtility method autocomplete()

    def autocomplete(self):
        """
        Output completion suggestions for BASH.

        The output of this function is passed to BASH's `COMREPLY` variable and
        treated as completion suggestions. `COMREPLY` expects a space
        separated string as the result.

        The `COMP_WORDS` and `COMP_CWORD` BASH environment variables are used
        to get information about the cli input. Please refer to the BASH
        man-page for more information about this variables.

        Subcommand options are saved as pairs. A pair consists of
        the long option string (e.g. '--exclude') and a boolean
        value indicating if the option requires arguments. When printing to
        stdout, an equal sign is appended to options which require arguments.

        Note: If debugging this function, it is recommended to write the debug
        output in a separate file. Otherwise the debug output will be treated
        and formatted as potential completion suggestions.
        """
        # Don't complete if user hasn't sourced bash_completion file.
        """如果没有 DJANGO_AUTO_COMPLETE 这个环境变量的话 那么就不会执行这个函数"""
        if 'DJANGO_AUTO_COMPLETE' not in os.environ:
            return

        cwords = os.environ['COMP_WORDS'].split()[1:]
        """cwords就是从系统中获取到用户输入的指令 然后split成一个列表 并且从第二个位置开始遍历"""
        cword = int(os.environ['COMP_CWORD'])
        """cword的作用是表示当前有几个词 用户输入的词的个数"""

        try:
            curr = cwords[cword - 1]
        except IndexError:
            curr = ''

        subcommands = [*get_commands(), 'help']
        """subcommands = 获取到django的所有command"""
        options = [('--help', False)]

        # subcommand
        if cword == 1:
        """如果当前用户只输入了一个词 那么就会打印出所有列表指令中开头是这个词的指令(lambda) 并打印出来"""
            print(' '.join(sorted(filter(lambda x: x.startswith(curr), subcommands))))
        # subcommand options
        # special case: the 'help' subcommand has no options
        elif cwords[0] in subcommands and cwords[0] != 'help':
            subcommand_cls = self.fetch_command(cwords[0])
            # special case: add the names of installed apps to options
            if cwords[0] in ('dumpdata', 'sqlmigrate', 'sqlsequencereset', 'test'):
                try:
                    app_configs = apps.get_app_configs()
                    # Get the last part of the dotted path as the app name.
                    options.extend((app_config.label, 0) for app_config in app_configs)
                except ImportError:
                    # Fail silently if DJANGO_SETTINGS_MODULE isn't set. The
                    # user will find out once they execute the command.
                    pass
            parser = subcommand_cls.create_parser('', cwords[0])
            options.extend(
                (min(s_opt.option_strings), s_opt.nargs != 0)
                for s_opt in parser._actions if s_opt.option_strings
            )
            # filter out previously specified options from available options
            prev_opts = {x.split('=')[0] for x in cwords[1:cword - 1]}
            options = (opt for opt in options if opt[0] not in prev_opts)

            # filter options by current input
            options = sorted((k, v) for k, v in options if k.startswith(curr))
            for opt_label, require_arg in options:
                # append '=' to options which require args
                if require_arg:
                    opt_label += '='
                print(opt_label)
        # Exit code of the bash completion function is never passed back to
        # the user, so it's safe to always exit with 0.
        # For more details see #25420.
        sys.exit(0)

```
* **os<br> [python 源码](https://github.com/python/cpython/blob/3.8/Lib/os.py) | [python docs](https://docs.python.org/3/library/os.html)**
**这个是python较为底层的逻辑 主要负责和系统交互**

**os.getcwd获取当前工作的环境路径**![image](http://note.youdao.com/yws/res/22922/WEBRESOURCE32eb077dd85a08755bb4d8eb782f152e)
**os.path.join()表示拼接路径 只不过路径中间由'/'分割**
```python
import os 

Path1 = 'home'
Path2 = 'develop'
Path3 = 'code'

Path10 = Path1 + Path2 + Path3
Path20 = os.path.join(Path1,Path2,Path3)
print 'Path10 = ',Path10
print 'Path20 = ',Path20 

result:
Path10 =  homedevelopcode
Path20 =  home/develop/code
```
**os.walk()<br>**
**os walk(path)会遍历path目录下的所有目录和文件 并且有三个参数:<br>**

1. **root表示当前的path目录**
2. **dirs表示当前目录下的所有文件夹名(不包括子文件夹)**
3. **files表示当前目录下的所有文件名(不包括子文件)**

* **django-admin verbosity<br>**
**[django docs](https://docs.djangoproject.com/en/3.0/ref/django-admin/#cmdoption-verbosity)**
![image](http://note.youdao.com/yws/res/22947/WEBRESOURCEfe361fc997529ed50f529d688f6faddf)

