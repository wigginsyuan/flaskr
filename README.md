# flaskr
Step 1: 数据库模式
首先我们要创建数据库模式。对于这个应用仅一张表就足够了，而且我们只想支持 SQLite ，所以很简单。 只要把下面的内容放入一个名为 schema.sql 的文件，文件置于刚才创建的 flaskr 文件夹中：

drop table if exists entries;
create table entries (
  id integer primary key autoincrement,
  title string not null,
  text string not null
);
这个模式由一个称为 entries 的单表构成，在这个表中每行包含一个 id ，一个 title 和一个 text。 id 是一个自增的整数而且是主键，其余两个为非空的字符串。

Step 2: 应用设置代码
现在我们已经有了数据库模式了，我们可以创建应用的模块了。让我们称为 flaskr.py ，并 放置于 flaskr 文件夹中。对于初学者来说，我们会添加所有需要的导入像配置的章节中一样。对于小应用，直接把配置放在主模块里，正如我们现在要做的一样，是可行的。然而一个更干净的解决方案就是单独创建 .ini 或者 .py 文件接着加载或者导入里面的值。

在 flaskr.py 中:

# all the imports
import sqlite3
from flask import Flask, request, session, g, redirect, url_for, \
     abort, render_template, flash

# configuration
DATABASE = '/tmp/flaskr.db'
DEBUG = True
SECRET_KEY = 'development key'
USERNAME = 'admin'
PASSWORD = 'default'
下一步我们能够创建真正的应用，接着用同一文件(在 flaskr.py 中)中的配置初始化:

# create our little application :)
app = Flask(__name__)
app.config.from_object(__name__)
from_object() 将会寻找给定的对象(如果它是一个字符串，则会导入它)， 搜寻里面定义的全部大写的变量。在我们的这种情况中，配置文件就是我们上面写的几行代码。 你也可以将他们分别存储到多个文件。

通常，从配置文件中加载配置是一个好的主意。这是 from_envvar() 所做的， 用它替换上面的 from_object()

app.config.from_envvar('FLASKR_SETTINGS', silent=True)
这种方法我们可以设置一个名为 FLASKR_SETTINGS 环境变量来设定一个配置文件载入后是否覆盖默认值。 静默开关告诉 Flask 不去关心这个环境变量键值是否存在。

secret_key 是需要为了保持客户端的会话安全。明智地选择该键，使得它难以猜测，尽可能复杂。 调试标志启用或禁用交互式调试。决不让调试模式在生产系统中启动，因为它将允许用户在服务器上执行代码！

我们还添加了一个轻松地连接到指定数据库的方法，这个方法用于在请求时打开一个连接，并且在交互式 Python shell 和脚本中也能使用。这对以后很方便。

def connect_db():
    return sqlite3.connect(app.config['DATABASE'])
最后如果我们想要把那个文件当做独立应用来运行，我们只需在服务器启动文件的末尾添加这一行:

if __name__ == '__main__':
    app.run()
如此我们便可以顺利开始运行这个应用，使用如下命令:

python flaskr.py
你将会看到一个信息，信息提示你服务器启动的地址，这个地址你能够访问到的。

当你在浏览器中访问服务器获得一个404页面无法找到的错误时，是因为我们还没有任何视图。我们之后再来关注这些。首先我们应该让数据库工作起来。

Step 3: 创建数据库
如前面所述，Flaskr 是一个数据库驱动的应用程序，准确地来说，Flaskr 是一个使用关系数据库系统的应用程序。 这样的系统需要一个模式告诉它们如何存储信息。因此在首次启动服务器之前，创建数据库模式是很重要的。

可以通过管道把 schema.sql 作为 sqlite 3 命令的输入来创建这个模式，命令如下:

sqlite3 /tmp/flaskr.db < schema.sql
这种方法的缺点是需要安装 sqlite 3 命令，而并不是每个系统都有安装。而且你必须提供数据库的路径，否则将报错。添加一个函数来对初始化数据库是个不错的想法。

如果你想要这么做，首先你必须从 contextlib 包中导入 contextlib.closing() 函数。并且在 flaskr.py 文件中添加如下的内容:

from contextlib import closing
接着我们可以创建一个称为 init_db 函数，该函数用来初始化数据库。为此我们可以使用之前定义的 connect_db 函数。 只要在 connect_db 函数下添加这样的函数:

def init_db():
    with closing(connect_db()) as db:
        with app.open_resource('schema.sql') as f:
            db.cursor().executescript(f.read())
        db.commit()
closing() 助手函数允许我们在 with 块中保持数据库连接可用。 应用对象的 open_resource() 方法在其方框外也支持这个功能， 因此可以在 with 块中直接使用。这个函数从资源位置（你的 flaskr 文 件夹）中打开一个文件，并且允许你读取它。我们在这里用它在数据库连接上执行一个脚本。

当我们连接到数据库时会得到一个数据库连接对象（这里命名它为 db ），这个对象提供给我们一个数据库指针。指针上有一个可以执行完整脚本的方法。最后我们不显式地提交更改， SQLite 3 或者其它事务数据库不会这么做。

现在可以在 Python shell 里创建数据库，导入并调用刚才的函数:

>>> from flaskr import init_db
>>> init_db()
Troubleshooting
如果你后面得到一个表不能找到的异常，请检查你是否调用了 init_db 函数以及你的表名是否正确 (例如： singular vs. plural)。

Step 4: 请求数据库连接
现在我们知道了怎样建立数据库连接以及在脚本中使用这些连接，但是我们如何能优雅地在请求中这么做？ 所有我们的函数中需要数据库连接，因此在请求之前初始化它们，在请求结束后自动关闭他们就很有意义。

Flask 允许我们使用 before_request()，after_request() 和 teardown_request() 装饰器来实现这个功能:

@app.before_request
def before_request():
    g.db = connect_db()

@app.teardown_request
def teardown_request(exception):
    g.db.close()
使用 before_request() 装饰器的函数会在请求之前被调用而且不带参数。使用 after_request() 装饰器的函数会在请求之后被调用且传入将要发给客户端的响应。 它们必须返回那个响应对象或是不同的响应对象。但当异常抛出时，它们不一定会被执行， 这时可以使用 teardown_request() 装饰器。它装饰的函数将在响应构造后执行， 并不允许修改请求，返回的值会被忽略。如果在请求已经被处理的时候抛出异常，它会被传递到每个函数， 否则会传入一个 None 。

我们把当前的数据库连接保存在 Flask 提供的 g 特殊对象中。这个对象只能保存一次请求的信息， 并且在每个函数里都可用。不要用其它对象来保存信息，因为在多线程环境下将不可行。特殊的对象 g 在后台有一些神 奇的机制来保证它在做正确的事情。

Step 5: 视图函数
现在数据库连接已经工作我们可以开始编写视图函数。我们需要四个视图函数：

显示条目
这个视图显示所有存储在数据库中的条目。它监听者应用的根地址以及将会从数据库中查询标题和内容。 id值最大的条目（最新的条目）将在最前面。从游标返回的行是按 select 语句中声明的列组织的元组。 对于像我们这样的小应用是足够的，但是你可能要把它们转换成字典。如果你对如何转换成字典感兴趣的话， 请查阅 简化查询 例子。

视图函数将会把条目作为字典传入 show_entries.html 模版以及返回渲染结果:

@app.route('/')
def show_entries():
    cur = g.db.execute('select title, text from entries order by id desc')
    entries = [dict(title=row[0], text=row[1]) for row in cur.fetchall()]
    return render_template('show_entries.html', entries=entries)
添加新条目
这个视图允许登录的用户添加新的条目。它只回应 POST 请求，实际的表单是显示在 show_entries 页面。 如果一些工作正常的话，我们用 flash() 向下一个请求闪现一条信息并且跳转回 show_entries 页:

@app.route('/add', methods=['POST'])
def add_entry():
    if not session.get('logged_in'):
        abort(401)
    g.db.execute('insert into entries (title, text) values (?, ?)',
                 [request.form['title'], request.form['text']])
    g.db.commit()
    flash('New entry was successfully posted')
    return redirect(url_for('show_entries'))
注意我们这里检查用户登录情况( logged_in 键存在会话中，并且为 True )。

安全提示
确保像上面例子中一样，使用问号标记来构建 SQL 语句。否则，当你使用格式化字符串构建 SQL 语句时， 你的应用容易遭受 SQL 注入。 更多的信息请看 在 Flask 中使用 SQLite 3 。
登录和注销
这些函数是用于用户登录以及注销。依据在配置中的值登录时检查用户名和密码并且在会话中设置 logged_in 键值。 如果用户成功登录，logged_in 键值被设置成 True ，并跳转回 show_entries 页。此外，会有消息闪现来提示用户登入成功。 如果发生一个错误，模板会通知，并提示重新登录:

@app.route('/login', methods=['GET', 'POST'])
def login():
    error = None
    if request.method == 'POST':
        if request.form['username'] != app.config['USERNAME']:
            error = 'Invalid username'
        elif request.form['password'] != app.config['PASSWORD']:
            error = 'Invalid password'
        else:
            session['logged_in'] = True
            flash('You were logged in')
            return redirect(url_for('show_entries'))
    return render_template('login.html', error=error)
另一方面，注销函数从会话中移除了 logged_in 键值。这里我们使用一个大绝招：如果你使用字典的 pop() 方法并传入第二个参数（默认）， 这个方法会从字典中删除这个键，如果这个键不存在则什么都不做。这很有用，因为 我们不需要检查用户是否已经登入。

@app.route('/logout')
def logout():
    session.pop('logged_in', None)
    flash('You were logged out')
    return redirect(url_for('show_entries'))
    
Step 6: 模版
现在我们应该开始编写模版。如果我们现在请求 URLs，我们将会得到一个 Flask 无法找到模版的异常。 模版使用 Jinja2 语言以及默认开启自动转义。这就意味着除非你使用 Markup 标记或在模板中使用 |safe 过滤器， 否则 Jinja2 会确保特殊字符比如 < 或 > 被转义成等价的 XML 实体。

我们也使用模版继承使得在网站的所有页面中重用布局成为可能。

请把如下的模版放入 templates 文件夹：

layout.html
这个模板包含 HTML 主体结构，标题和一个登录链接（或者当用户已登入则提供注销）。如果有闪现信息的话它也将显示 闪现信息。{% block body %} 块能够被子模版中的同样名字( body )的块替代。

session 字典在模版中同样可用的，你能用它检查用户是否登录。注意在 Jinja 中你可以访问不存在的对象/字典属性或成员， 如同下面的代码， 即便 'logged_in' 键不存在，仍然可以正常工作：

<!doctype html>
<title>Flaskr</title>
<link rel=stylesheet type=text/css href="{{ url_for('static', filename='style.css') }}">
<div class=page>
  <h1>Flaskr</h1>
  <div class=metanav>
  {% if not session.logged_in %}
    <a href="{{ url_for('login') }}">log in</a>
  {% else %}
    <a href="{{ url_for('logout') }}">log out</a>
  {% endif %}
  </div>
  {% for message in get_flashed_messages() %}
    <div class=flash>{{ message }}</div>
  {% endfor %}
  {% block body %}{% endblock %}
</div>
show_entries.html
这个模版继承了上面的 layout.html 模版用来显示信息。注意 for 遍历了所有我们用 render_template() 函数传入的信息。我们同样告诉表单提交到 add_entry 函数且使用 HTTP 的 POST 方法：

{% extends "layout.html" %}
{% block body %}
  {% if session.logged_in %}
    <form action="{{ url_for('add_entry') }}" method=post class=add-entry>
      <dl>
        <dt>Title:
        <dd><input type=text size=30 name=title>
        <dt>Text:
        <dd><textarea name=text rows=5 cols=40></textarea>
        <dd><input type=submit value=Share>
      </dl>
    </form>
  {% endif %}
  <ul class=entries>
  {% for entry in entries %}
    <li><h2>{{ entry.title }}</h2>{{ entry.text|safe }}
  {% else %}
    <li><em>Unbelievable.  No entries here so far</em>
  {% endfor %}
  </ul>
{% endblock %}
login.html
最后是登录模板，基本上只显示一个允许用户登录的表单：

{% extends "layout.html" %}
{% block body %}
  <h2>Login</h2>
  {% if error %}<p class=error><strong>Error:</strong> {{ error }}{% endif %}
  <form action="{{ url_for('login') }}" method=post>
    <dl>
      <dt>Username:
      <dd><input type=text name=username>
      <dt>Password:
      <dd><input type=password name=password>
      <dd><input type=submit value=Login>
    </dl>
  </form>
{% endblock %}

Step 7: 添加样式
现在其他一切都正常工作，是时候给应用添加些样式。只要在我们之前创建的 static 文件夹中新建一个称为 style.css 的样式表:

body            { font-family: sans-serif; background: #eee; }
a, h1, h2       { color: #377BA8; }
h1, h2          { font-family: 'Georgia', serif; margin: 0; }
h1              { border-bottom: 2px solid #eee; }
h2              { font-size: 1.2em; }

.page           { margin: 2em auto; width: 35em; border: 5px solid #ccc;
                  padding: 0.8em; background: white; }
.entries        { list-style: none; margin: 0; padding: 0; }
.entries li     { margin: 0.8em 1.2em; }
.entries li h2  { margin-left: -1em; }
.add-entry      { font-size: 0.9em; border-bottom: 1px solid #ccc; }
.add-entry dl   { font-weight: bold; }
.metanav        { text-align: right; font-size: 0.8em; padding: 0.3em;
                  margin-bottom: 1em; background: #fafafa; }
.flash          { background: #CEE5F5; padding: 0.5em;
                  border: 1px solid #AACBE2; }
.error          { background: #F0D6D6; padding: 0.5em; }
