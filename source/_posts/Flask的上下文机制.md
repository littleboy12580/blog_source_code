---
layout: post
title: "Flask的上下文机制"
date: 2017-07-15 21:39
comments: true
tags:
	- Flask
	- 技术
---

<!-- more -->
## 什么是上下文
Flask的官方文档里并没有对上下文的概念做出一个直观地解释；所以在网上找了很多解释，最后还是觉得知乎的一个回答解释的最直观：

> 每一段程序都有很多外部变量。只有像Add这种简单的函数才是没有外部变量的。一旦你的一段程序有了外部变量，这段程序就不完整，不能独立运行。你为了使他们运行，就要给所有的外部变量一个一个写一些值进去。这些值的集合就叫上下文

举例来说，我们在flask应用里收到一个请求，应用的视图函数需要知道它执行所需要的一些请求信息（请求的url，参数，方法），
如果这个请求涉及到数据库的话还需要知道一些应用信息（例如数据库初始化的一些信息），这样它才能正确的运行。

一个比较直接的做法是把这些信息封装成一个对象，作为参数传给视图函数。但这种做法会导致所有的视图函数都需要添加对应的参数，而有的函数内部其实并没有用到这些参数。

为了解决这个问题，Flask把这些信息弄成了类似全局变量的东西，不过与全局变量不同的是这个线程安全的（在多线程服务器中，通过线程池处理不同客户的不同请求，当收到请求后，会选一个线程进行处理，
请求的临时对象（也就是上下文）会保存在该线程对应的全局变量中（通过线程 ID 区分），这样即不干扰其他线程，又使得所有线程都可以访问）

## Flask中的上下文实现
Flask中有两种上下文，请求上下文（Request Context）和应用上下文（App Context）
查看Flask源码目录下的 __init__.py 文件，可以发现上下文这些变量实际上是在 globals.py 中定义，然后只是在 __init__.py 中引入的，其定义如下：
```python
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)


def _lookup_app_object(name):
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
    return getattr(top, name)


def _find_app():
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
    return top.app


# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
g = LocalProxy(partial(_lookup_app_object, 'g'))
```
从上面的代码可以看出，request context演化出了request和session这两个变量；
而app context则演化出了current_app和g这两个变量。

### LocalStack与Local
两个上下文都是LocalStack的实例，我们先来看看LocalStack的实现源码：
```python
class LocalStack(object):

    """This class works similar to a :class:`Local` but keeps a stack
    of objects instead.  This is best explained with an example::

        >>> ls = LocalStack()
        >>> ls.push(42)
        >>> ls.top
        42
        >>> ls.push(23)
        >>> ls.top
        23
        >>> ls.pop()
        23
        >>> ls.top
        42

    They can be force released by using a :class:`LocalManager` or with
    the :func:`release_local` function but the correct way is to pop the
    item from the stack after using.  When the stack is empty it will
    no longer be bound to the current context (and as such released).

    By calling the stack without arguments it returns a proxy that resolves to
    the topmost item on the stack.

    .. versionadded:: 0.6.1
    """

    def __init__(self):
        self._local = Local()

    # 用来清空当前线程的栈数据
    def __release_local__(self):
        self._local.__release_local__()

    def _get__ident_func__(self):
        return self._local.__ident_func__

    def _set__ident_func__(self, value):
        object.__setattr__(self._local, '__ident_func__', value)
    __ident_func__ = property(_get__ident_func__, _set__ident_func__)
    del _get__ident_func__, _set__ident_func__

    # 返回当前线程栈顶元素的代理对象
    def __call__(self):
        def _lookup():
            rv = self.top
            if rv is None:
                raise RuntimeError('object unbound')
            return rv
        return LocalProxy(_lookup)

    # push、pop 和 top 三个方法实现了栈的操作
    # 栈的数据是保存在 self._local.stack 属性中
    def push(self, obj):
        """Pushes a new item to the stack"""
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        """Removes the topmost item from the stack, will return the
        old value or `None` if the stack was already empty.
        """
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    @property
    def top(self):
        """The topmost item on the stack.  If the stack is empty,
        `None` is returned.
        """
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```
根据源码的解释与实现可以看出，LocalStack相当于Local类的栈实现，所以我们来看看Local的实现源码：

```python
class Local(object):
    # 我理解的__slots__就是用来限制给类的实例添加新的属性
    # 不过这个地方应该是它真正的用法，在创建大量实例的时候更加节省内存
    # 原理是使用__slots__的属性，其属性名会直接和类关联，而不再和实例关联，不会有属性值对应字典了
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    # 下面三个方法实现了属性的访问、设置和删除。
    # 内部都调用 `self.__ident_func__` 获取当前线程的 id，然后再访问对应的内部字典。
    # 如果访问的属性不存在，会在当前线程新建改属性
    # 如果删除的属性不存在，会抛出 AttributeError。
    # 这样，外部用户看到的就是它在访问实例的属性，完全不知道多线程切换的实现
    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```
Local 对象内部的数据都是保存在 __storage__ 属性的，这个属性变量是个嵌套的字典：map[ident]map[key]value。
最外面字典 key 是线程的 identity，value 是另外一个字典，这个内部字典就是用户自定义的 key-value 键值对。
用户访问实例的属性，就变成了访问内部的字典，外面字典的 key 是自动关联的。
__ident_func__ 相当于线程的 get_ident，从而获取当前代码所在线程或者协程的 id。

理解了上述代码可以知道，上下文是一个栈的结构，它会把当前线程的请求都保存在栈里，等使用的时候再从里面读取
至于说为什么要用栈，我的理解是为了那种多个Flask应用一起工作的情况（虽然目前我还没碰到）


### 关于LocalProxy
从最开始的init文件可以看出，四个上下文变量都用到了LocalProxy，先来看看LocalProxy的实现；
```python
class LocalProxy(object):
    __slots__ = ('__local', '__dict__', '__name__')

    def __init__(self, local, name=None):
        # 前面双下划线的属性，会保存到 _ClassName__variable 中
        # 所以这里通过 “_LocalProxy__local” 设置的值，后面可以通过 self.__local 来获取
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)

    def _get_current_object(self):
        """Return the current object.  This is useful if you want the real
        object behind the proxy at a time for performance reasons or because
        you want to pass the object into a different context.
        """
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

    @property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError('__dict__')

    def __repr__(self):
        try:
            obj = self._get_current_object()
        except RuntimeError:
            return '<%s unbound>' % self.__class__.__name__
        return repr(obj)

    def __bool__(self):
        try:
            return bool(self._get_current_object())
        except RuntimeError:
            return False

    def __unicode__(self):
        try:
            return unicode(self._get_current_object())  # noqa
        except RuntimeError:
            return repr(self)

    def __dir__(self):
        try:
            return dir(self._get_current_object())
        except RuntimeError:
            return []

    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value

    def __delitem__(self, key):
        del self._get_current_object()[key]

    if PY2:
        __getslice__ = lambda x, i, j: x._get_current_object()[i:j]

        def __setslice__(self, i, j, seq):
            self._get_current_object()[i:j] = seq

        def __delslice__(self, i, j):
            del self._get_current_object()[i:j]

    # 重写了所有的魔术方法，使具体的操作都转发给了代理的对象
    __setattr__ = lambda x, n, v: setattr(x._get_current_object(), n, v)
    __delattr__ = lambda x, n: delattr(x._get_current_object(), n)
    __str__ = lambda x: str(x._get_current_object())
    __lt__ = lambda x, o: x._get_current_object() < o
    __le__ = lambda x, o: x._get_current_object() <= o
    __eq__ = lambda x, o: x._get_current_object() == o
    __ne__ = lambda x, o: x._get_current_object() != o
    __gt__ = lambda x, o: x._get_current_object() > o
    __ge__ = lambda x, o: x._get_current_object() >= o
    __cmp__ = lambda x, o: cmp(x._get_current_object(), o)  # noqa
    __hash__ = lambda x: hash(x._get_current_object())
    __call__ = lambda x, *a, **kw: x._get_current_object()(*a, **kw)
    __len__ = lambda x: len(x._get_current_object())
    __getitem__ = lambda x, i: x._get_current_object()[i]
    __iter__ = lambda x: iter(x._get_current_object())
    __contains__ = lambda x, i: i in x._get_current_object()
    __add__ = lambda x, o: x._get_current_object() + o
    __sub__ = lambda x, o: x._get_current_object() - o
    __mul__ = lambda x, o: x._get_current_object() * o
    __floordiv__ = lambda x, o: x._get_current_object() // o
    __mod__ = lambda x, o: x._get_current_object() % o
    __divmod__ = lambda x, o: x._get_current_object().__divmod__(o)
    __pow__ = lambda x, o: x._get_current_object() ** o
    __lshift__ = lambda x, o: x._get_current_object() << o
    __rshift__ = lambda x, o: x._get_current_object() >> o
    __and__ = lambda x, o: x._get_current_object() & o
    __xor__ = lambda x, o: x._get_current_object() ^ o
    __or__ = lambda x, o: x._get_current_object() | o
    __div__ = lambda x, o: x._get_current_object().__div__(o)
    __truediv__ = lambda x, o: x._get_current_object().__truediv__(o)
    __neg__ = lambda x: -(x._get_current_object())
    __pos__ = lambda x: +(x._get_current_object())
    __abs__ = lambda x: abs(x._get_current_object())
    __invert__ = lambda x: ~(x._get_current_object())
    __complex__ = lambda x: complex(x._get_current_object())
    __int__ = lambda x: int(x._get_current_object())
    __long__ = lambda x: long(x._get_current_object())  # noqa
    __float__ = lambda x: float(x._get_current_object())
    __oct__ = lambda x: oct(x._get_current_object())
    __hex__ = lambda x: hex(x._get_current_object())
    __index__ = lambda x: x._get_current_object().__index__()
    __coerce__ = lambda x, o: x._get_current_object().__coerce__(x, o)
    __enter__ = lambda x: x._get_current_object().__enter__()
    __exit__ = lambda x, *a, **kw: x._get_current_object().__exit__(*a, **kw)
    __radd__ = lambda x, o: o + x._get_current_object()
    __rsub__ = lambda x, o: o - x._get_current_object()
    __rmul__ = lambda x, o: o * x._get_current_object()
    __rdiv__ = lambda x, o: o / x._get_current_object()
    if PY2:
        __rtruediv__ = lambda x, o: x._get_current_object().__rtruediv__(o)
    else:
        __rtruediv__ = __rdiv__
    __rfloordiv__ = lambda x, o: o // x._get_current_object()
    __rmod__ = lambda x, o: o % x._get_current_object()
    __rdivmod__ = lambda x, o: x._get_current_object().__rdivmod__(o)
    __copy__ = lambda x: copy.copy(x._get_current_object())
    __deepcopy__ = lambda x, memo: copy.deepcopy(x._get_current_object(), memo)
```
LocalProxy 是一个 Local 对象的代理，负责把所有对自己的操作转发给内部的 Local 对象。
LocalProxy 的构造函数介绍一个 callable 的参数，这个 callable 调用之后需要返回一个 Local 实例
当对LocalProxy属性进行访问的时候，它永远都是先执行_get_current_object，该函数就是执行我们写入的回调函数得到访问对象，
比如current_app就是得到_app_ctx_stack.top.app对象。最后再对得到的对象进行属性访问

### 为什么要用LocalProxy
我个人的理解就是为了调用更方便，举个例子，有下面的代码：
```python
l = LocalStack()
l.push(RequestContext())
```
RequestContext里有个属性是request，现在我如果要访问request这个属性，我需要执行l.top.request，
而且由于l.top.request是随着线程进入退出而一直出入栈动态变化的，我还不能直接把它赋给一个变量，而是需要每次都去执行一次，这就很不方便了
此时使用LocalProxy就能很好的解决这个问题

## RequestContext与AppContext
了解了上下文的整体结构，我们可以知道flask获取上下文取的就是_request_ctx_stack/_app_ctx_stack栈顶的变量，
而这两个变量就是RequestContext与AppContext，那么它们是何时被推到栈里的呢？
可以看看Flask的运行过程：
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run()
```
应用启动的代码是 app.run() ，这个方法的代码如下：
```python
def run(self, host=None, port=None, debug=None, **options):
    """Runs the application on a local development server."""
    from werkzeug.serving import run_simple

    if host is None:
        host = '127.0.0.1'
    if port is None:
        server_name = self.config['SERVER_NAME']
        if server_name and ':' in server_name:
            port = int(server_name.rsplit(':', 1)[1])
        else:
            port = 5000

    # 调用 werkzeug.serving 模块的 run_simple 函数，传入收到的参数
    # 注意第三个参数传进去的是 self，也就是要执行的 web application
    try:
        run_simple(host, port, self, **options)
    finally:
        self._got_first_request = False
```
使用的是werkzeug的run_simple，根据wsgi规范，app是一个接口，并接受两个参数,即，application(environ, start_response)
在run_wsgi的run_wsgi我们可以清晰的看到调用过程：
```python
def run_wsgi(self):
    environ = self.make_environ()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    reraise(*exc_info)
            finally:
                exc_info = None
        elif headers_set:
            raise AssertionError('Headers already set')
        headers_set[:] = [status, response_headers]
        return write

    def execute(app):
        application_iter = app(environ, start_response)
        #environ是为了给request传递请求的
        #start_response主要是增加响应头和状态码，最后需要werkzeug发送请求
        try:
            for data in application_iter: #根据wsgi规范，app返回的是一个序列
                write(data) #发送结果
            if not headers_sent:
                write(b'')
        finally:
            if hasattr(application_iter, 'close'):
                application_iter.close()
            application_iter = None

    try:
        execute(self.server.app)
    except (socket.error, socket.timeout) as e:
        pass
```
flask中通过定义__call__方法适配wsgi规范：
```python
class Flask(_PackageBoundObject):
    def __call__(self, environ, start_response):
        """Shortcut for :attr:`wsgi_app`."""
        return self.wsgi_app(environ, start_response)

    def wsgi_app(self, environ, start_response):
        ctx = self.request_context(environ)
        #这个ctx就是我们所说的同时有request,session属性的上下文
        ctx.push()
        error = None
        try:
            try:
                response = self.full_dispatch_request()
            except Exception as e:
                error = e
                response = self.make_response(self.handle_exception(e))
            return response(environ, start_response)
        finally:
            if self.should_ignore_error(error):
                error = None
            ctx.auto_pop(error)

    def request_context(self, environ):
        return RequestContext(self, environ)
```
可以看出，当一个请求来的时候会调用ctx.push()

现在看看RequestContext的源码：
```python
class RequestContext(object):
    """The request context contains all request relevant information.  It is
    created at the beginning of the request and pushed to the
    `_request_ctx_stack` and removed at the end of it.  It will create the
    URL adapter and request object for the WSGI environment provided.
    """

    def __init__(self, app, environ, request=None):
        self.app = app
        if request is None:
            # 根据环境变量创建request
            request = app.request_class(environ)
        self.request = request
        self.url_adapter = app.create_url_adapter(self.request)
        self.match_request()

    # 实现路由的匹配
    def match_request(self):
        """Can be overridden by a subclass to hook into the matching
        of the request.
        """
        try:
            url_rule, self.request.view_args = \
                self.url_adapter.match(return_rule=True)
            self.request.url_rule = url_rule
        except HTTPException as e:
            self.request.routing_exception = e

    def push(self):
        # 将ctx push进 _request_ctx_stack
        app_ctx = _app_ctx_stack.top
        if app_ctx is None or app_ctx.app != self.app:
            app_ctx = self.app.app_context()
            app_ctx.push()
            self._implicit_app_ctx_stack.append(app_ctx)
        else:
            self._implicit_app_ctx_stack.append(None)

        _request_ctx_stack.push(self)

        # 压栈后还会保存 session 的信息
        self.session = self.app.open_session(self.request)
        if self.session is None:
            self.session = self.app.make_null_session()

    def pop(self, exc=_sentinel):
        """Pops the request context and unbinds it by doing that.  This will
        also trigger the execution of functions registered by the
        :meth:`~flask.Flask.teardown_request` decorator.
        """
        app_ctx = self._implicit_app_ctx_stack.pop()

        try:
            clear_request = False
            if not self._implicit_app_ctx_stack:
                self.app.do_teardown_request(exc)

                request_close = getattr(self.request, 'close', None)
                if request_close is not None:
                    request_close()
                clear_request = True
        finally:
            rv = _request_ctx_stack.pop()

            # get rid of circular dependencies at the end of the request
            # so that we don't require the GC to be active.
            if clear_request:
                rv.request.environ['werkzeug.request'] = None

            # Get rid of the app as well if necessary.
            if app_ctx is not None:
                app_ctx.pop(exc)

    def auto_pop(self, exc):
        if self.request.environ.get('flask._preserve_context') or \
           (exc is not None and self.app.preserve_context_on_exception):
            self.preserved = True
            self._preserved_exc = exc
        else:
            self.pop(exc)

    def __enter__(self):
        self.push()
        return self

    def __exit__(self, exc_type, exc_value, tb):
        self.auto_pop(exc_value)
```
push 操作就是把该请求的ApplicationContext和RequestContext有关的信息保存到对应的栈上，
如果_app_ctx_stack栈顶不存在或者不是当前请求所在app，则会自动创建一个新的app context


所以这就是上下文的整个工作流程了，
每次有请求过来的时候，flask 会先创建当前线程或者进程需要处理的两个重要上下文对象，把它们保存到隔离的栈里面，
这样视图函数进行处理的时候就能直接从栈上获取这些信息

