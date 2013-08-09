<!---
Don't edit this file manually! Instead you should generate it by using:
    wiki2markdown.pl HttpLuaModule.wiki
-->

名称
======

ngx_lua - 在Nging中嵌入强大的Lua

''此模块属于第三方扩展，不包含在Nginx源码发布包中。 参阅 [安装](http://wiki.nginx.org/HttpLuaModule#Installation).

状态
======

模块持续更新中，可用于生产环境。

版本
======

本文对应的是 ngx_lua [v0.8.6](https://github.com/chaoslawful/lua-nginx-module/tags)版本，发布于2013年8月6日.

示例
======

    # 设置纯Lua编写的外部库的查找路径 (';;' 是默认查找路径):
    lua_package_path '/foo/bar/?.lua;/blah/?.lua;;';
 
    # 设置用C编写的外部库的查找路径 (也可以使用 ';;'):
    lua_package_cpath '/bar/baz/?.so;/blah/blah/?.so;;';
 
    server {
        location /inline_concat {
            # 使用默认的 MIME 类型:
            default_type 'text/plain';
 
            set $a "hello";
            set $b "world";
            # 内嵌Lua脚本
            set_by_lua $res "return ngx.arg[1]..ngx.arg[2]" $a $b;
            echo $res;
        }
 
        location /rel_file_concat {
            set $a "foo";
            set $b "bar";
            # 使用相对于$ngx_prefix的脚本路径
            # $ngx_prefix/conf/concat.lua 的内容:
            #
            #    返回 ngx.arg[1]..ngx.arg[2]
            #
            set_by_lua_file $res conf/concat.lua $a $b;
            echo $res;
        }
 
        location /abs_file_concat {
            set $a "fee";
            set $b "baz";
            # 直接使用绝对路径 
            set_by_lua_file $res /usr/nginx/conf/concat.lua $a $b;
            echo $res;
        }
 
        location /lua_content {
            # 使用默认的 MIME 类型:
            default_type 'text/plain';
 
            content_by_lua "ngx.say('Hello,world!')";
        }
 
         location /nginx_var {
            # 使用默认的 MIME 类型:
            default_type 'text/plain';
 
            # 试试访问 /nginx_var?a=hello,world
            content_by_lua "ngx.print(ngx.var['arg_a'], '\\n')";
        }
 
        location /request_body {
             # 强制读取 request body (默认关闭)
             lua_need_request_body on;
             client_max_body_size 50k;
             client_body_buffer_size 50k;
 
             content_by_lua 'ngx.print(ngx.var.request_body)';
        }
 
        # 在Lua中使用subrequests进行透明的非阻塞I/O
        location /lua {
            # 使用默认的 MIME 类型:
            default_type 'text/plain';
 
            content_by_lua '
                local res = ngx.location.capture("/some_other_location")
                if res.status == 200 then
                    ngx.print(res.body)
                end';
        }
 
        # GET /recur?num=5
        location /recur {
            # 使用默认的 MIME 类型:
            default_type 'text/plain';
 
            content_by_lua '
               local num = tonumber(ngx.var.arg_num) or 0

               if num > 50 then
                   ngx.say("num too big")
                   return
               end

               ngx.say("num is: ", num)
 
               if num > 0 then
                   res = ngx.location.capture("/recur?num=" .. tostring(num - 1))
                   ngx.print("status=", res.status, " ")
                   ngx.print("body=", res.body)
               else
                   ngx.say("end")
               end
               ';
        }
 
        location /foo {
            rewrite_by_lua '
                res = ngx.location.capture("/memc",
                    { args = { cmd = "incr", key = ngx.var.uri } }
                )
            ';
 
            proxy_pass http://blah.blah.com;
        }
 
        location /blah {
            access_by_lua '
                local res = ngx.location.capture("/auth")
 
                if res.status == ngx.HTTP_OK then
                    return
                end
 
                if res.status == ngx.HTTP_FORBIDDEN then
                    ngx.exit(res.status)
                end
 
                ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
            ';
 
            # proxy_pass/fastcgi_pass/postgres_pass/...
        }
 
        location /mixed {
            rewrite_by_lua_file /path/to/rewrite.lua;
            access_by_lua_file /path/to/access.lua;
            content_by_lua_file /path/to/content.lua;
        }
 
        # 在代码路径中使用 nginx 变量
        # 注意：nginx 变量中的内容必须被小心过滤
        # 否则会产生巨大的安全漏洞!        location ~ ^/app/(.+) {
                content_by_lua_file /path/to/lua/app/root/$1.lua;
        }
 
        location / {
           lua_need_request_body on;
 
           client_max_body_size 100k;
           client_body_buffer_size 100k;
 
           access_by_lua '
               -- 检查客户端IP是否在黑名单里
               if ngx.var.remote_addr == "132.5.72.3" then
                   ngx.exit(ngx.HTTP_FORBIDDEN)
               end
 
               -- 检查request body中是否包含恶意词或敏感词
               if ngx.var.request_body and
                        string.match(ngx.var.request_body, "fsck")
               then
                   return ngx.redirect("/terms_of_use.html")
               end
 
               -- 测试通过
           ';
 
           # proxy_pass/fastcgi_pass/etc settings
        }
    }


说明
======

此模块将Lua（标准的Lua 5.1或 [LuaJIT 2.0](http://luajit.org/luajit.html)）嵌入Nginx, 利用了Nginx的subrequest，把强大的Lua协程（Lua coroutines）集成进Nginx的事件模型。

与[Apache's mod_lua](http://httpd.apache.org/docs/2.3/mod/mod_lua.html) 和 [Lighttpd's mod_magnet](http://redmine.lighttpd.net/wiki/1/Docs:ModMagnet) 不同, 只要使用[Nginx API for Lua](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)来处理到upstream服务如MySQL，PostgreSQL，Memcached, Redis或upstream web service的请求, 使用本模块运行的Lua代码在处理网络请求时可以是 *100% 非阻塞的*。 

至少下列Lua库和Nginx模块可以与ngx_lua模块一同使用:

* [lua-resty-memcached](https://github.com/agentzh/lua-resty-memcached)
* [lua-resty-mysql](https://github.com/agentzh/lua-resty-mysql)
* [lua-resty-redis](https://github.com/agentzh/lua-resty-redis)
* [lua-resty-dns](https://github.com/agentzh/lua-resty-dns)
* [lua-resty-upload](https://github.com/agentzh/lua-resty-upload)
* [ngx_memc](http://wiki.nginx.org/HttpMemcModule)
* [ngx_postgres](https://github.com/FRiCKLE/ngx_postgres)
* [ngx_redis2](http://wiki.nginx.org/HttpRedis2Module)
* [ngx_redis](http://wiki.nginx.org/HttpRedisModule)
* [ngx_proxy](http://wiki.nginx.org/HttpProxyModule)
* [ngx_fastcgi](http://wiki.nginx.org/HttpFastcgiModule)

几乎所有的Nginx模块都可以通过 [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) 或 [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi)来使用ngx_lua模块。但推荐使用 `lua-resty-*` 库而不是创建子请求来访问Nginx upstream模块，因为前者通常有高得多的灵活性和内存效率。

在单个nginx worker进程中Lua解释器或LuaJIT实例为所有请求所共享，但是请求上下文使用了轻量的Lua coroutine进行了隔离。

被加载的Lua模块保存在nginx worker进程一级，即使在高负载情况下也只有很小的内存开销。

命令
======

lua_code_cache
--------------
**语法:** *lua_code_cache on | off*

**默认值:** *lua_code_cache on*

**可包含于:** *main, server, location, location if*

为 [set_by_lua_file](http://wiki.nginx.org/HttpLuaModule#set_by_lua_file),
[content_by_lua_file](http://wiki.nginx.org/HttpLuaModule#content_by_lua_file), [rewrite_by_lua_file](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_file), and
[access_by_lua_file](http://wiki.nginx.org/HttpLuaModule#access_by_lua_file) 打开或关闭Lua代码缓存, 同时强制为每个请求重新加载Lua模块。

在[set_by_lua_file](http://wiki.nginx.org/HttpLuaModule#set_by_lua_file),
[content_by_lua_file](http://wiki.nginx.org/HttpLuaModule#content_by_lua_file), [access_by_lua_file](http://wiki.nginx.org/HttpLuaModule#access_by_lua_file),
和 [rewrite_by_lua_file](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_file)中引用的Lua文件不会被缓存， Lua的 `package.loaded` table会在每个请求的入口被清空 (所以 Lua 模块也不会被缓存). 有了它，开发者可以采用象开发php那样修改-刷新即可 

但是请注意，在nginx.conf中内置写入的 Lua 代码，如 [set_by_lua](http://wiki.nginx.org/HttpLuaModule#set_by_lua), [content_by_lua](http://wiki.nginx.org/HttpLuaModule#content_by_lua),
[access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua), 和 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua)中的，将“总是”被缓存，因为只有 Nginx 配置文件解析器能够正确解析 `nginx.conf`
文件，重新加载配置文件的方法是发送 `HUP` 信号或重启 Nginx.

还有，*_by_lua_file中使用 `dofile` 或 `loadfile`加载的Lua文件决不会被缓存. 为了保证代码被缓存，你要么用 [init_by_lua](http://wiki.nginx.org/HttpLuaModule#init_by_lua)
或 [init_by_lua_file](http://wiki.nginx.org/HttpLuaModule#init-by_lua_file) 命令来加载文件或将这些Lua文件写成真正的Lua模块，并使用 `require` 来加载.

ngx_lua 模块目前不支持Apache `mod_lua` module 中有的 `stat` 模式，但我们计划在将来实现它。

禁止代码缓存在生产环境中是非常不推荐的，对整体的性能有很大的负面影响，只应用于开发阶段。另外，如果禁止了代码缓存，在并发请求中重新加载模块经常会导致race condition。

lua_regex_cache_max_entries
---------------------------
**语法:** *lua_regex_cache_max_entries &lt;num&gt;*

**默认值:** *lua_regex_cache_max_entries 1024*

**可包含于:** *http*

设置在worker进程级别允许缓存的编译后正则表达式的条数

[ngx.re.match](http://wiki.nginx.org/HttpLuaModule#ngx.re.match), [ngx.re.gmatch](http://wiki.nginx.org/HttpLuaModule#ngx.re.gmatch), [ngx.re.sub](http://wiki.nginx.org/HttpLuaModule#ngx.re.sub), and [ngx.re.gsub](http://wiki.nginx.org/HttpLuaModule#ngx.re.gsub)中使用的正则表达式，如果设置了 `o` (即“仅编译一次”）模式，将在这个缓存中进行缓存。

默认的条数是1024，当达到这个上限时，新的正则表达式将不被缓存 (就象没有设置 `o` 一样) ，同时在 `error.log` 文件中会记录一条且仅记录一条warning:


    2011/08/27 23:18:26 [warn] 31997#0: *1 lua exceeding regex cache max entries (1024), ...


不要为运行时动态生成的正则表达式设置`o`模式，（或是为 [ngx.re.sub](http://wiki.nginx.org/HttpLuaModule#ngx.re.sub) 和 [ngx.re.gsub](http://wiki.nginx.org/HttpLuaModule#ngx.re.gsub) 提供`replace` 字符串参数) 以避免超限。

lua_regex_match_limit
---------------------
**语法:** *lua_regex_match_limit &lt;num&gt;*

**默认值:** *lua_regex_match_limit 0*

**可包含于:** *http*

指定在执行[[#ngx.re.matchngx.re API]]时PCRE库所使用的 "match limit" 参数。引自 PCRE manpage： "这个上限... 是用来限制正则表达式中允许的backtracking的最大数目"

当达到这个上限时, 在Lua一方，[[#ngx.re.matchngx.re API]]会返回 "pcre_exec() failed: -8" 错误信息。

如果此值设为0, 将使用编译 PCRE 库时的默认“match limit”值. 0也是这个命令的默认值.

本命令在 `v0.8.5` 版本中引入.

lua_package_path
----------------

**语法:** *lua_package_path &lt;lua-style-path-str&gt;*

**默认值:** *The content of LUA_PATH environ variable or Lua's compiled-in defaults.*

**可包含于:** *main*

设置由 [[#set_by_luaset_by_lua]],
[[#content_by_luacontent_by_lua]]或其它指令写入的Lua脚本代码使用的Lua模块查找路径. 路径字符串使用标准的Lua路径格式，, `;;`可用于表示原始的查找路径。

从`v0.5.0rc29`版本开始, 特殊标记 `$prefix` 或 `${prefix}` 可以在路径字符串中用来表示 `server prefix` 的路径，它通常可由在Nginx服务器启动时使用 `-p PATH` 命令行参数来得到。

lua_package_cpath
-----------------

**语法:** *lua_package_cpath &lt;lua-style-cpath-str&gt;*

**默认值:** *环境变量LUA_CPATH的内容或 Lua编译时指定的默认值.*

**可包含于:** *main*

设置由[[#set_by_luaset_by_lua]],
[[#content_by_luacontent_by_lua]]或其它指定写入的Lua脚本代码使用的C模块查找路径。cpath字符串使用标准的Lua cpath格式， `;;`
可用来表示原始的cpath.

从`v0.5.0rc29`版本开始, 特殊标记 `$prefix` 或 `${prefix}` 可以在路径字符串中用来表示 `server prefix` 的路径，它通常可由在Nginx服务器启动时使用 `-p PATH` 命令行参数来得到。

init_by_lua
-----------

**语法:** *init_by_lua &lt;lua-script-str&gt;*

**可包含于:** *http*

**执行阶段:** *loading-config*

在Nginx主进程（如果存在的话）加载Nginx配置文件时在全局Lua虚拟机级别运行由参数`<lua-script-str>`指定的Lua代码

如果 Nginx 收到`HUP` 信号，开始重新加载配置文件, Lua虚拟机将被重新创建， `init_by_lua`会在新的Lua虚拟机中被重新运行。

通常你可以使用这个hook在服务器启动时来注册（真正的）Lua全局变量 或预加载Lua模块。以下是预加载 Lua 的例子:


    init_by_lua 'cjson = require "cjson"';

    server {
        location = /api {
            content_by_lua '
                ngx.say(cjson.encode({dog = 5, cat = 6}))
            ';
        }
    }


也可以在这个阶段对 [lua_shared_dict](http://wiki.nginx.org/HttpLuaModule#lua_shared_dict) 共享存储进行初始化。例如：


    lua_shared_dict dogs 1m;

    init_by_lua '
        local dogs = ngx.shared.dogs;
        dogs:set("Tom", 56)
    ';

    server {
        location = /api {
            content_by_lua '
                local dogs = ngx.shared.dogs;
                ngx.say(dogs:get("Tom"))
            ';
        }
    }


但请注意， [lua_shared_dict](http://wiki.nginx.org/HttpLuaModule#lua_shared_dict)的共享存储在重新加载配置文件（如通过`HUP`信号）时不会被清空。所以如果在这种情况下如果你不希望在`init_by_lua`代码中重新对共享存储进行初始化，你只需要在共享存储中设置一个标记，并在 `init_by_lua` 代码中检查这个标记.

由于在这个上下文中的Lua代码是在Nginx fork它的worker进程（如果有的话）之前运行的，所以这里加载和数据或代码会在work进程之间享受很多操作系统都提供的 [Copy-on-write (COW)](http://en.wikipedia.org/wiki/Copy-on-write)特性，能够节省大量内存。

这个上下文中只支持很少一部分 [Nginx API for Lua](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua) :

* Logging APIs: [ngx.log](http://wiki.nginx.org/HttpLuaModule#ngx.log) and [print](http://wiki.nginx.org/HttpLuaModule#print),
* Shared Dictionary API: [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

如果将来有更多用户要求的话，会为这个上下文添加更多的Nginx API支持。

基本上你可以在这个上下文中安全地使用进行阻塞I/O的Lua库，因为在启动阶段阻塞主进程是完全ok的。 Nginx 内核自己在加载配置的阶段也用阻塞 I/O (至少在解析 upstream 主机名时) .

要非常小心避免Lua代码中潜在的安全漏洞，因为Nginx主进程经常用 `root` 帐号来运行.

本命令在 `v0.5.5` 版本中引入。

init_by_lua_file
----------------

**语法:** *init_by_lua_file &lt;path-to-lua-script-file&gt;*

**可包含于:** *http*

**执行阶段:** *loading-config*

类似 [init_by_lua](http://wiki.nginx.org/HttpLuaModule#init_by_lua), 区别在于 `<path-to-lua-script-file>` 可以包含 Lua 代码或是 [Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support)。

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

本命令在 `v0.5.5` 版本中引入。

set_by_lua
----------

**语法:** *set_by_lua $res &lt;lua-script-str&gt; [$arg1 $arg2 ...]*

**可包含于:** *server, server if, location, location if*

**执行阶段:** *server-rewrite, rewrite*

执行 `<lua-script-str>` 中的代码，可选参数 `$arg1 $arg2 ...`, 返回字符串输出到 `$res`中. 
`<lua-script-str>`中的代码可以调用 [API](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)并能从 `ngx.arg` table中获取输入参数 (下标从 `1`开始递增).

这条命令是被设计用来执行短的，能很快结束的代码块，在它执行时Nginx事件循环是被阻塞的。所以应避免在这里编写耗时较长的代码序列。

注意在这个上下文中以下 API 函数是被禁止的:

* Output API 函数(例如, [ngx.say](http://wiki.nginx.org/HttpLuaModule#ngx.say) and [ngx.send_headers](http://wiki.nginx.org/HttpLuaModule#ngx.send_headers))
* Control API 函数(例如, [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit)) 
* Subrequest API 函数(例如, [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) and [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi))
* Cosocket API 函数(例如, [ngx.socket.tcp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp) and [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)).

另外，注意这条命令每次只能写入一个Nginx变量。但是可以使用 [ngx.var.VARIABLE](http://wiki.nginx.org/HttpLuaModule#ngx.var.VARIABLE) 来绕过这个限制。


    location /foo {
        set $diff ''; # 必须先定义 $diff 变量
 
        set_by_lua $sum '
            local a = 32
            local b = 56
 
            ngx.var.diff = a - b;  -- 直接写入 $diff 
            return a + b;          -- 照常返回 $sum 值
        ';
 
        echo "sum = $sum, diff = $diff";
    }


这条命令可以随意与 [HttpRewriteModule](http://wiki.nginx.org/HttpRewriteModule), [HttpSetMiscModule](http://wiki.nginx.org/HttpSetMiscModule), 和[HttpArrayVarModule](http://wiki.nginx.org/HttpArrayVarModule) 模块中的命令混合使用. 所有的命令按照它们在配置文件中出现的次序执行。


    set $foo 32;
    set_by_lua $bar 'tonumber(ngx.var.foo) + 1';
    set $baz "bar: $bar";  # $baz == "bar: 33"


从 `v0.5.0rc29` 起, 本命令中的`<lua-script-str>`中禁止使用Nginx 变量 interpolation ，因此,(`$`)可以直接使用.

这条命令需要 [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit) 模块.

set_by_lua_file
---------------
**语法:** *set_by_lua_file $res &lt;path-to-lua-script-file&gt; [$arg1 $arg2 ...]*

**可包含于:** *server, server if, location, location if*

**执行阶段:** *server-rewrite, rewrite*

同 [set_by_lua](http://wiki.nginx.org/HttpLuaModule#set_by_lua), 区别在于 `<path-to-lua-script-file>` 可以包含 Lua 代码, 或者, 从 `v0.5.0rc32` 版本开始, 可以包含 [Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support)。 

本命令的`<path-to-lua-script-file>`参数支持Nginx 变量 interpolation。但是要特别小心防止注入攻击。

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

如果Lua代码缓存被打开 (这是默认行为), 用户代码在第一个请求中被加载并缓存，每次Lua代码文件被修改后，需要重新加载Nginx配置文件。在开发阶段，Lua代码缓存可以通过在`nginx.conf`中设置 [lua_code_cache](http://wiki.nginx.org/HttpLuaModule#lua_code_cache) `off` 来避免重新加载Nginx配置文件。

这条命令需要 [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit) 模块.

content_by_lua
--------------

**语法:** *content_by_lua &lt;lua-script-str&gt;*

**可包含于:** *location, location if*

**执行阶段:** *content*

作为一个 "content handler"，为每个请求执行 `<lua-script-str>`中的代码. 
Lua代码可以调用 [API](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua) 并以一个拥有独立的全局环境的新生成的coroutine（即一个sandbox）的形式运行.

在同一个location中不要同时使用本命令和其它content handler命令。例如， 这条命令和 [proxy_pass](http://wiki.nginx.org/HttpProxyModule#proxy_pass) 命令不允许在同一个location中使用。

content_by_lua_file
-------------------

**语法:** *content_by_lua_file &lt;path-to-lua-script-file&gt;*

**可包含于:** *location, location if*

**执行阶段:** *content*

同 [content_by_lua](http://wiki.nginx.org/HttpLuaModule#content_by_lua), 区别在于 `<path-to-lua-script-file>` 可以包含或, 从`v0.5.0rc32` 版本开始, 可以包含 [Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support).

为了更灵活， `<path-to-lua-script-file>` 中允许使用Nginx变量. 但这种做法会带来风险，通常并不推荐。

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

如果Lua代码缓存被打开 (这是默认行为), 用户代码在第一个请求中被加载并缓存，每次Lua代码文件被修改后，需要重新加载Nginx配置文件。在开发阶段，Lua代码缓存可以通过在`nginx.conf`中设置 [lua_code_cache](http://wiki.nginx.org/HttpLuaModule#lua_code_cache) `off` 来避免重新加载Nginx配置文件。

rewrite_by_lua
--------------

**语法:** *rewrite_by_lua &lt;lua-script-str&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *rewrite tail*

作为一个 rewrite phase handler，为每个请求执行 `<lua-script-str>` 中的代码。Lua代码可以调用 [API](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua) 并以一个拥有独立的全局环境的新生成的coroutine（即一个sandbox）的形式运行.

注意这个handler总是在标准 [HttpRewriteModule](http://wiki.nginx.org/HttpRewriteModule) *之后*执行 . 以下代码会得到预期的结果:


       location /foo {
           set $a 12; # 创建并初始化 $a
           set $b ""; # 创建并初始化 $b
           rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
           echo "res = $b";
       }


因为`set $a 12` 和 `set $b ""` 在[[#rewrite_by_luarewrite_by_lua]] *之前* 执行 .

而下面的代码将不能得到预期的结果:


    ?location /foo {
    ?set $a 12; # 创建并初始化 $a
    ?set $b ''; # 创建并初始化 $b
    ?rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
    ?if ($b = '13') {
    ?rewrite ^ /bar redirect;
    ?break;
    ?}
    ?    ?echo "res = $b";
    ?}


因为 `if` 虽然写在[[#rewrite_by_luarewrite_by_lua]]之后，但还是在  [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) *之前*执行。

正确的作法是这样的:


    location /foo {
        set $a 12; # 创建并初始化 $a
        set $b ''; # 创建并初始化 $b
        rewrite_by_lua '
            ngx.var.b = tonumber(ngx.var.a) + 1
            if tonumber(ngx.var.b) == 13 then
                return ngx.redirect("/bar");
            end
        ';
 
        echo "res = $b";
    }


注意 [ngx_eval](http://www.grid.net.ru/nginx/eval.en.html) 模块可以用 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua)来实现. 例如,


    location / {
        eval $res {
            proxy_pass http://foo.com/check-spam;
        }
 
        if ($res = 'spam') {
            rewrite ^ /terms-of-use.html redirect;
        }
 
        fastcgi_pass ...;
    }


可以用 ngx_lua 这样实现:


    location = /check-spam {
        internal;
        proxy_pass http://foo.com/check-spam;
    }
 
    location / {
        rewrite_by_lua '
            local res = ngx.location.capture("/check-spam")
            if res.body == "spam" then
                return ngx.redirect("/terms-of-use.html")
            end
        ';
 
        fastcgi_pass ...;
    }


与其它的 rewrite phase handlers一样, [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua)也运行在 subrequests 中.

注意在 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) handler中调用`ngx.exit(ngx.OK)`时, nginx请求处理控制流仍然会继续执行 content handler. To terminate the current request from within a 要在 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) handler中终止请求处理, 如果是成功退出，要调用 [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit)将状态码设置为 >= 200 (`ngx.HTTP_OK`)  < 300 (`ngx.HTTP_SPECIAL_RESPONSE`) 的值，如果是失败退出调用 `ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)` (或类似的值).

如果使用了 [HttpRewriteModule](http://wiki.nginx.org/HttpRewriteModule) 的 [rewrite](http://wiki.nginx.org/HttpRewriteModule#rewrite) 命令来修改 URI 和初始 location re-lookups (内部重定向), 则当前location中的任何 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) 或 [rewrite_by_lua_file](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_file) 代码都不会被执行。例如,


    location /foo {
        rewrite ^ /bar;
        rewrite_by_lua 'ngx.exit(503)';
    }
    location /bar {
        ...
    }


此处 `ngx.exit(503)` 不会被执行. 如果写成 `rewrite ^ /bar last` 也是同样结果，因为这也会类似地引发内部重定向相反，如果使用了 `break` , 就不会有内部重定向， `rewrite_by_lua` 将被执行.

除非[rewrite_by_lua_no_postpone](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_no_postpone)被设成on， `rewrite_by_lua`总是在`rewrite`请求处理阶段的最后执行。

rewrite_by_lua_file
-------------------

**语法:** *rewrite_by_lua_file &lt;path-to-lua-script-file&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *rewrite tail*

同[rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua), 区别在于 `<path-to-lua-script-file>` 可以包含或, 从 `v0.5.0rc32` 版本开始, 可以包含[Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support).

为了更灵活， `<path-to-lua-script-file>` 中允许使用Nginx变量. 但这种做法会带来风险，通常并不推荐。

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

如果Lua代码缓存被打开 (这是默认行为), 用户代码在第一个请求中被加载并缓存，每次Lua代码文件被修改后，需要重新加载Nginx配置文件。在开发阶段，Lua代码缓存可以通过在`nginx.conf`中设置 [lua_code_cache](http://wiki.nginx.org/HttpLuaModule#lua_code_cache) `off` 来避免重新加载Nginx配置文件。

除非[rewrite_by_lua_no_postpone](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_no_postpone)被设成on， `rewrite_by_lua_file`总是在`rewrite`请求处理阶段的最后执行。

access_by_lua
-------------

**语法:** *access_by_lua &lt;lua-script-str&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *access tail*

作为一个 access phase handler运行，为每个请求执行 `<lua-script-str>` 中的代码.
Lua代码可以调用 [API](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua) 并以一个拥有独立的全局环境的新生成的coroutine（即一个sandbox）的形式运行.

注意这个handler总是在标准 [HttpAccessModule](http://wiki.nginx.org/HttpAccessModule) *之后*执行 . 以下代码会得到预期的结果:


    location / {
        deny    192.168.1.1;
        allow   192.168.1.0/24;
        allow   10.1.1.0/16;
        deny    all;
 
        access_by_lua '
            local res = ngx.location.capture("/mysql", { ... })
            ...
        ';
 
        # proxy_pass/fastcgi_pass/...
    }


如果客户端的IP地址在黑名单中，则请求将在[[#access_by_luaaccess_by_lua]]进行MySQL查询进行更复杂的认证之前被拒绝。

注意 [ngx_auth_request](http://mdounin.ru/hg/ngx_http_auth_request_module/) 模块可以用 [[#access_by_luaaccess_by_lua]]来实现.


    location / {
        auth_request /auth;
 
        # proxy_pass/fastcgi_pass/postgres_pass/...
    }


可以用 ngx_lua 这样实现:


    location / {
        access_by_lua '
            local res = ngx.location.capture("/auth")
 
            if res.status == ngx.HTTP_OK then
                return
            end
 
            if res.status == ngx.HTTP_FORBIDDEN then
                ngx.exit(res.status)
            end
 
            ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
        ';
 
        # proxy_pass/fastcgi_pass/postgres_pass/...
    }


与其它的 rewrite phase handlers一样, [[#access_by_luaaccess_by_lua]]也*不会*运行在 subrequests 中.

注意在 [[#access_by_luaaccess_by_lua]] handler中调用`ngx.exit(ngx.OK)`时, nginx请求处理控制流仍然会继续执行 content handler. 要在 [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua) 处理器中终止请求处理, 如果是成功退出，要调用 [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit)将状态码设置为 >= 200 (`ngx.HTTP_OK`)  < 300 (`ngx.HTTP_SPECIAL_RESPONSE`) 的值，如果是失败退出调用 `ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)` (或类似的值).

access_by_lua_file
------------------

**语法:** *access_by_lua_file &lt;path-to-lua-script-file&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *access tail*

同 [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua), 区别在于 `<path-to-lua-script-file>` 可以包含 Lua 代码, 或者, 从 `v0.5.0rc32` 版本开始, 可以包含 [Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support)。

为了更灵活， `<path-to-lua-script-file>` 中允许使用Nginx变量. 但这种做法会带来风险，通常并不推荐。

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

如果Lua代码缓存被打开 (这是默认行为), 用户代码在第一个请求中被加载并缓存，每次Lua代码文件被修改后，需要重新加载Nginx配置文件。在开发阶段，Lua代码缓存可以通过在`nginx.conf`中设置 [lua_code_cache](http://wiki.nginx.org/HttpLuaModule#lua_code_cache) `off` 来避免重新加载Nginx配置文件。

header_filter_by_lua
--------------------

**语法:** *header_filter_by_lua &lt;lua-script-str&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *output-header-filter*

用 `<lua-script-str>` 中的代码来定义一个输出头过滤器。

注意在这个上下文中以下 API 函数是被禁止的:

* Output API 函数(例如, [ngx.say](http://wiki.nginx.org/HttpLuaModule#ngx.say) and [ngx.send_headers](http://wiki.nginx.org/HttpLuaModule#ngx.send_headers))
* Control API 函数(例如, [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit)) 
* Subrequest API 函数(例如, [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) and [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi))
* Cosocket API 函数(例如, [ngx.socket.tcp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp) and [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)).

下面是在Lua 头过滤器中重写（如果没有就添加）一个响应头的例子


    location / {
        proxy_pass http://mybackend;
        header_filter_by_lua 'ngx.header.Foo = "blah"';
    }


本命令在 `v0.2.1rc20` 版本中引入.

header_filter_by_lua_file
-------------------------

**语法:** *header_filter_by_lua_file &lt;path-to-lua-script-file&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *output-header-filter*

同 [[#header_filter_by_luaheader_filter_by_lua]], 区别在于 `<path-to-lua-script-file>` 可以包含 Lua 代码, 或者, 从 `v0.5.0rc32` 版本开始, 可以包含 [Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support)。

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

本命令在 `v0.2.1rc20` 版本中引入.

body_filter_by_lua
------------------

**语法:** *body_filter_by_lua &lt;lua-script-str&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *output-body-filter*

用 `<lua-script-str>` 中的代码来定义一个输出体过滤器。

输入参数块由 [ngx.arg](http://wiki.nginx.org/HttpLuaModule#ngx.arg)[1] (作为Lua string)传入， 标记标识响应体数据流结束的"eof"由 [ngx.arg](http://wiki.nginx.org/HttpLuaModule#ngx.arg)[2] (作为Lua boolean)传入.

在内部实现里， "eof" 标记其实是Nginx缓冲区链中的 `last_buf` (主请求) 或 `last_in_chain` (子请求) 标记(在`v0.7.14`之前的版本中, "eof" 标记对子请求无效.)

可以用过执行下面的Lua语句来立即终止响应数据流:


    return ngx.ERROR


这将会截断响应体，通常会造成不完整并且是无效的响应

Lua 代码可以修改 [ngx.arg](http://wiki.nginx.org/HttpLuaModule#ngx.arg)[1] 将自己修改后的输入数据块作为Lua string或装有string的Lua table送到下游的Nginx输出体过滤器中。例如，要将转换响应体中的所有小写字母，可以写:


    location / {
        proxy_pass http://mybackend;
        body_filter_by_lua 'ngx.arg[1] = string.upper(ngx.arg[1])';
    }


如果将`ngx.arg[1]`设置成 `nil` 或空的Lua string , 将不会有数据发往下游 Nginx输出过滤器。

类似的，也可以将[[#ngx.argngx.arg]][2]的值设为新的boolean值来修改 "eof" 标记。例如,


    location /t {
        echo hello world;
        echo hiya globe;

        body_filter_by_lua '
            local chunk = ngx.arg[1]
            if string.match(chunk, "hello") then
                ngx.arg[2] = true  -- 新的 eof
                return
            end

            -- 丢弃剩下的块数据
            ngx.arg[1] = nil
        ';
    }


请求 `GET /t` 将产生以下输出


    hello world


当过滤器看到包含 "hello" 的数据块, 就立即将 "eof" 标记设为true, 产生被截断但仍然是有效的响应内容。

如果Lua 代码可能改变响应体的长度, 就总是需要用头过滤器来清除 `Content-Length` 响应头 (如果有的话)来保证流输出的正确性, 例如


    location /foo {
        # fastcgi_pass/proxy_pass/...

        header_filter_by_lua 'ngx.header.content_length = nil';
        body_filter_by_lua 'ngx.arg[1] = string.len(ngx.arg[1]) .. "\\n"';
    }


注意在这个上下文中以下 API 函数是被禁止的:

* Output API 函数(例如, [ngx.say](http://wiki.nginx.org/HttpLuaModule#ngx.say) and [ngx.send_headers](http://wiki.nginx.org/HttpLuaModule#ngx.send_headers))
* Control API 函数(例如, [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit)) 
* Subrequest API 函数(例如, [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) and [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi))
* Cosocket API 函数(例如, [ngx.socket.tcp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp) and [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)).

Nginx 输入过滤器对单个请求可能被调用多次，这是因为响应体可能是分段发送的所以，在单个请求的生命周期中本命令中的代码也可能运行多次。

本命令在 `v0.5.0rc32` 版本中引入.

body_filter_by_lua_file
-----------------------

**语法:** *body_filter_by_lua_file &lt;path-to-lua-script-file&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *output-body-filter*

同 [body_filter_by_lua](http://wiki.nginx.org/HttpLuaModule#body_filter_by_lua), 区别在于 `<path-to-lua-script-file>` 可以包含 Lua 代码, 或者, 从 `v0.5.0rc32` 版本开始, 可以包含 [Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support)。

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

本命令在 `v0.5.0rc32` 版本中引入.

log_by_lua
----------

**语法:** *log_by_lua &lt;lua-script-str&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *log*

在请求处理的 `log` 阶段运行`<lua-script-str>`中的lua代码. 它不会覆盖当前的访问日志，而是在记录访问日志之后运行.

注意在这个上下文中以下 API 函数是被禁止的:

* Output API 函数(例如, [ngx.say](http://wiki.nginx.org/HttpLuaModule#ngx.say) and [ngx.send_headers](http://wiki.nginx.org/HttpLuaModule#ngx.send_headers))
* Control API 函数(例如, [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit)) 
* Subrequest API 函数(例如, [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) and [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi))
* Cosocket API 函数(例如, [ngx.socket.tcp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp) and [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)).

以下是收集 [$upstream_response_time](http://wiki.nginx.org/HttpUpstreamModule#.24upstream_response_time) 平均值的例子:


    lua_shared_dict log_dict 5M;

    server {
        location / {
            proxy_pass http://mybackend;

            log_by_lua '
                local log_dict = ngx.shared.log_dict
                local upstream_time = tonumber(ngx.var.upstream_response_time)

                local sum = log_dict:get("upstream_time-sum") or 0
                sum = sum + upstream_time
                log_dict:set("upstream_time-sum", sum)

                local newval, err = log_dict:incr("upstream_time-nb", 1)
                if not newval and err == "not found" then
                    log_dict:add("upstream_time-nb", 0)
                    log_dict:incr("upstream_time-nb", 1)
                end
            ';
        }

        location = /status {
            content_by_lua '
                local log_dict = ngx.shared.log_dict
                local sum = log_dict:get("upstream_time-sum")
                local nb = log_dict:get("upstream_time-nb")
    
                if nb and sum then
                    ngx.say("average upstream response time: ", sum / nb,
                            " (", nb, " reqs)")
                else
                    ngx.say("no data yet")
                end
            ';
        }
    }


本命令在 `v0.5.0rc31` 版本中引入.

log_by_lua_file
---------------

**语法:** *log_by_lua_file &lt;path-to-lua-script-file&gt;*

**可包含于:** *http, server, location, location if*

**执行阶段:** *log*

同[log_by_lua](http://wiki.nginx.org/HttpLuaModule#log_by_lua), 区别在于 `<path-to-lua-script-file>` 可以包含或, 从 `v0.5.0rc32` 版本开始, 可以包含[Lua/LuaJIT 字节码](http://wiki.nginx.org/HttpLuaModule#Lua/LuaJIT_bytecode_support).

如果指定的是相对路径如 `foo/bar.lua` , 它们将被加上 `server prefix` 转换成绝对路径。`server prefix`可由在Nginx服务器启动时使用`-p PATH` 命令行参数来得到。

本命令在 `v0.5.0rc31` 版本中引入.

lua_need_request_body
---------------------

**语法:** *lua_need_request_body &lt;on|off&gt;*

**默认值:** *off*

**可包含于:** *main | server | location*

**执行阶段:** *depends on usage*

决定在运行 rewrite/access/access_by_lua* 之前是否强制读取请求体数据. Nginx内核默认不读取请求体，如果需要请求体数据, 就需要将本命令置为 `on` ，或者在Lua代码中调用 [ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body) 函数.

要在 [$request_body](http://wiki.nginx.org/HttpCoreModule#.24request_body) 变量中读取请求体数据, 
[client_body_buffer_size](http://wiki.nginx.org/HttpCoreModule#client_body_buffer_size) 必须与 [client_max_body_size](http://wiki.nginx.org/HttpCoreModule#client_max_body_size) 的值相同. 因为如果内容长度超过了[client_body_buffer_size](http://wiki.nginx.org/HttpCoreModule#client_body_buffer_size) 但小于 [client_max_body_size](http://wiki.nginx.org/HttpCoreModule#client_max_body_size), Nginx 将会把数据缓存到存盘上的临时文件中， [$request_body](http://wiki.nginx.org/HttpCoreModule#.24request_body)变量中将是空值.

如果当前的 location 包含[rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) 或 [rewrite_by_lua_file](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_file) 命令,
那么请求体将在 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) 或 [rewrite_by_lua_file](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_file) 代码运行之前被读取 (也是在
`rewrite` 阶段). 类似的，, 如果只设置了 [content_by_lua](http://wiki.nginx.org/HttpLuaModule#content_by_lua) , 请求体将在进行内容处理Lua 代码将要运行时被读取 (即, 请求体将在content阶段被读取).

推荐使用 [ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body) 和 [ngx.req.discard_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.discard_body) 函数来对请求体的读取过程进行精细控制。

这也适用于 [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua) 和 [access_by_lua_file](http://wiki.nginx.org/HttpLuaModule#access_by_lua_file).

lua_shared_dict
---------------

**语法:** *lua_shared_dict &lt;name&gt; &lt;size&gt;*

**默认值:** *no*

**可包含于:** *http*

**执行阶段:** *depends on usage*

声明一个共享内存区， `<name>`, 作为基于共享内存的 Lua 字典 `ngx.shared.<name>`的存储区.

`<size>` 参数支持象 `k` 和 `m`这样的大小单位:


    http {
        lua_shared_dict dogs 10m;
        ...
    }


详情见[ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT) .

本命令在 `v0.3.1rc22` 版本中引入.

lua_socket_connect_timeout
--------------------------

**语法:** *lua_socket_connect_timeout &lt;time&gt;*

**默认值:** *lua_socket_connect_timeout 60s*

**可包含于:** *http, server, location*

本命令控制 TCP/unix-domain socket 对象[connect](http://wiki.nginx.org/HttpLuaModule#tcpsock:connect) 方法中使用的默认超时时间， 可以用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 方法进行修改.

`<time>` 参数可以是整数，时间单位可选, 如`s` (秒), `ms` (毫秒), `m` (分钟). 默认的时间单位是 `s`, 即, "秒". 默认值是 `60s`.

本命令在 `v0.5.0rc1` 版本中引入.

lua_socket_send_timeout
-----------------------

**语法:** *lua_socket_send_timeout &lt;time&gt;*

**默认值:** *lua_socket_send_timeout 60s*

**可包含于:** *http, server, location*

本命令控制 TCP/unix-domain socket 对象[send](http://wiki.nginx.org/HttpLuaModule#tcpsock:send) 方法中使用的默认超时时间， 可以用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 方法进行修改.

`<time>` 参数可以是整数，时间单位可选, 如`s` (秒), `ms` (毫秒), `m` (分钟). 默认的时间单位是 `s`, 即, "秒". 默认值是 `60s`.

本命令在 `v0.5.0rc1` 版本中引入.

lua_socket_send_lowat
---------------------

**语法:** *lua_socket_send_lowat &lt;size&gt;*

**默认值:** *lua_socket_send_lowat 0*

**可包含于:** *http, server, location*

控制cosocket发送缓冲区的 `lowat` 值.

lua_socket_read_timeout
-----------------------

**语法:** *lua_socket_read_timeout &lt;time&gt;*

**默认值:** *lua_socket_read_timeout 60s*

**可包含于:** *http, server, location*

**执行阶段:** *depends on usage*

本命令控制 TCP/unix-domain socket 对象[receive](http://wiki.nginx.org/HttpLuaModule#tcpsock:receive) 方法中使用的默认超时时间， 可以用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 方法进行修改. 可以用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 方法进行修改.

`<time>` 参数可以是整数，时间单位可选, 如`s` (秒), `ms` (毫秒), `m` (分钟). 默认的时间单位是 `s`, 即, "秒". 默认值是 `60s`.

本命令在 `v0.5.0rc1` 版本中引入.

lua_socket_buffer_size
----------------------

**语法:** *lua_socket_buffer_size &lt;size&gt;*

**默认值:** *lua_socket_buffer_size 4k/8k*

**可包含于:** *http, server, location*

指定cosocket读操作的缓冲区大小。

这个缓冲区不需要大于能够一次装下所有的东西，因为 cosocket 支持 100% 无缓冲读取和解析. 所以即使是 `1` 字节的缓冲区大小也可以工作，只是性能会很糟糕。

本命令在 `v0.5.0rc1` 版本中引入.

lua_socket_pool_size
--------------------

**语法:** *lua_socket_pool_size &lt;size&gt;*

**默认值:** *lua_socket_pool_size 30*

**可包含于:** *http, server, location*

指定与每个远程服务器(由主机-端口号或unix domain socket文件路径标识)所关联的每个cosocket连接池的大小（连接数）上限。

默认每个连接池最多30个连接。

当连接池使用超限时, 池中最不经常使用的 (空闲) 连接将被关闭，为当前连接腾出空间。

注意cosocket连接池是每个nginx worker进程所有的而不是整个nginx服务器进程所有，所以这里所设置的上限也适用于每个单独的nginx worker 进程.

本命令在 `v0.5.0rc1` 版本中引入.

lua_socket_keepalive_timeout
----------------------------

**语法:** *lua_socket_keepalive_timeout &lt;time&gt;*

**默认值:** *lua_socket_keepalive_timeout 60s*

**可包含于:** *http, server, location*

本命令控制cosocket内置连接池中连接的默认最大空闲时间。如果超时, 空闲的连接将被关闭并从池中删除。可以用 cosocket 对象的' [setkeepalive](http://wiki.nginx.org/HttpLuaModule#tcpsock:setkeepalive) 方法来修改.

`<time>` 参数可以是整数，时间单位可选, 如`s` (秒), `ms` (毫秒), `m` (分钟). 默认的时间单位是 `s`, 即, "秒". 默认值是 `60s`.

本命令在 `v0.5.0rc1` 版本中引入.

lua_socket_log_errors
---------------------

**语法:** *lua_socket_log_errors on|off*

**默认值:** *lua_socket_log_errors on*

**可包含于:** *http, server, location*

本命令可以用来打开/关闭TCP或UDP cosocket出错时的错误日志。如果你的Lua代码中已经进行了正确的错误处理和记录日志，那么建议将此命令关闭，以避免在nginx错误日志文件中不必要的数据（这通常有很大的开销）。

本命令在 `v0.5.13` 版本中引入.

lua_http10_buffering
--------------------

**语法:** *lua_http10_buffering on|off*

**默认值:** *lua_http10_buffering on*

**可包含于:** *http, server, location, location-if*

打开/关闭 HTTP 1.0(或更低版本)的请求的自动响应缓冲。这个缓冲机制主要用于HTTP 1.0 keep-alive，基于正确的 `Content-Length` 响应头.

如果Lua代码在发送头（显式地通过[[#ngx.send_headersngx.send_headers]]或者隐式地通过第一个[[#ngx.sayngx.say]] 或 [[#ngx.printngx.print]]调用）之前显式地设置了 `Content-Length` 响应头，, 那么即使本命令被打开， 也不会进行HTTP 1.0 响应缓冲。

如果要以流的方式输出大的响应数据 (例如通过 [ngx.flush](http://wiki.nginx.org/HttpLuaModule#ngx.flush) 调用), 本命令必须关闭来使内存的使用最小化。

本命令默认值为 `on` .

本命令在 `v0.5.0rc19` 版本中引入.

rewrite_by_lua_no_postpone
--------------------------

**语法:** *rewrite_by_lua_no_postpone on|off*

**默认值:** *rewrite_by_lua_no_postpone off*

**可包含于:** *http*

控制是否禁止延后 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) 和[rewrite_by_lua_file](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_file) 命令，使其在 `rewrite` 请求处理阶段的最后运行. 默认情况下，本命令处于关闭状态，  Lua 代码在 `rewrite` 阶段的最后运行.

本命令在 `v0.5.0rc29` 版本中引入.

lua_transform_underscores_in_response_headers
---------------------------------------------

**语法:** *lua_transform_underscores_in_response_headers on|off*

**默认值:** *lua_transform_underscores_in_response_headers on*

**可包含于:** *http, server, location, location-if*

控制是否将 [ngx.header.HEADER](http://wiki.nginx.org/HttpLuaModule#ngx.header.HEADER) API中指定的响应头名称里的下划线 (`_`) 转换成减号 (`-`).

本命令在 `v0.5.0rc32` 版本中引入.

lua_check_client_abort
----------------------

**语法:** *lua_check_client_abort on|off*

**默认值:** *lua_check_client_abort off*

**可包含于:** *http, server, location, location-if*

本命令控制是否检测客户端连接过早中断。

如果本命令被打开， ngx_lua 模块将监视下游连接的过早连接关闭事件。当这种事件发生时，模块将调用用户提供的回调函数 (使用[ngx.on_abort](http://wiki.nginx.org/HttpLuaModule#ngx.on_abort)注册) ，如果用户没有提供回调函数，模块会简单地停止和清除所有在当前请求处理器中的Lua "轻量线程" 。

但是，根据当前的实现，如果客户端在Lua代码使用[[#ngx.req.socketngx.req.socket]]读取请求体数据之前就关闭的连接, ngx_lua模块既不会停止所有运行中的 "轻量线程" 也不会调用用户回调 (如果 [ngx.on_abort](http://wiki.nginx.org/HttpLuaModule#ngx.on_abort) 曾经被调用的话). 而是从对[ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)的读操作中返回，携带错误信息 "client aborted" 作为第二个返回值 (第一个返回值当然是 `nil`).

如果TCP keepalive被禁止, 需要依赖客户端来完美地关闭socket (发送 `FIN` 或类似的包). 对于 (软) 实时web应用, 强烈推荐为系统的TCP栈实现配置 [TCP keepalive](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html) 支持以便随时检测 "半开"TCP连接。

例如，在Linux上，你可以在`nginx.conf`文件中这样配置标准的 [listen](http://wiki.nginx.org/HttpCoreModule#listen) 命令:


    listen 80 so_keepalive=2s:2s:8;


在 FreeBSD 上, 你只能在系统级别配置 TCP keepalive, 如:

    # sysctl net.inet.tcp.keepintvl=2000
    # sysctl net.inet.tcp.keepidle=2000

本命令在 `v0.7.4` 版本中引入.

参阅 [ngx.on_abort](http://wiki.nginx.org/HttpLuaModule#ngx.on_abort).

lua_max_pending_timers
----------------------

**语法:** *lua_max_pending_timers &lt;count&gt;*

**默认值:** *lua_max_pending_timers 1024*

**可包含于:** *http*

控制未过期定时器数量的上限.

未过期定时器指尚未过期的定时器.

当定时器数量超限时, [ngx.timer.at](http://wiki.nginx.org/HttpLuaModule#ngx.timer.at) 调用将立即返回 `nil` 和错误信息 "too many pending timers".

本命令在 `v0.8.0` 版本中引入.

lua_max_running_timers
----------------------

**语法:** *lua_max_running_timers &lt;count&gt;*

**默认值:** *lua_max_running_timers 256*

**可包含于:** *http*

控制“正在运行的定时器“数量的上限.

正在运行的定时器指用户指定的回调函数仍在运行的定时器。

当数量超限时，, Nginx将停止运行最新过期的定时器并记录错误日志消息 "N lua_max_running_timers are not enough" ，其中 "N" 是本命令当前所设置的值。

本命令在 `v0.8.0` 版本中引入.

可供Lua调用的Nginx API
===========================
简介
------
各种 `*_by_lua` and `*_by_lua_file` 配置命令的作用是在`nginx.conf`文件内部提供访问Lua API的入口. 以下的 Nginx Lua API 只能由运行在这些配置命令上下文中的Lua代码进行调用。

这套API是以两个标准包 `ngx` and `ndk` 的形式暴露给Lua的. 这两个包在ngx_lua模块的默认全局作用域中， 在ngx_lua命令里总是可以访问到.

它们可以象这样被引入到外部 Lua 模块中:


    local say = ngx.say

    module(...)

    function foo(a) 
        say(a) 
    end


由于其所带来的各种不好的负作用，强烈不推荐使用 [package.seeall](http://www.lua.org/manual/5.1/manual.html#pdf-package.seeall) 标记。

也可以直接在外部Lua模块里require这两个包:


    local ngx = require "ngx"
    local ndk = require "ndk"


require 这两个包的能力始于 `v0.2.1rc19` 版本.

用户代码中的网络I/O操作只允许通过Nginx Lua API来进行，否则Nginx事件循环可能被阻塞，导致性能大降。对少量数据的磁盘操作可以用标准Lua `io` 库来完成，但是应尽可能避免对大文件的读取和写入，因为这会严重阻塞Nginx进程。为获得最好的性能，强烈推荐将网络和磁盘I/O委托给 Nginx's 子请求 (通过 [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) 或类似的方法)。

ngx.arg
-------
**语法:** *val = ngx.arg[index]*

**可包含于:** *set_by_lua*, body_filter_by_lua**

如果被用于 [set_by_lua](http://wiki.nginx.org/HttpLuaModule#set_by_lua) 或 [set_by_lua_file](http://wiki.nginx.org/HttpLuaModule#set_by_lua_file) 命令的上下文中, 这个table是只读的，装有配置命令的输入参数:


    value = ngx.arg[n]


示例


    location /foo {
        set $a 32;
        set $b 56;
 
        set_by_lua $res
            'return tonumber(ngx.arg[1]) + tonumber(ngx.arg[2])'
            $a $b;
 
        echo $sum;
    }


输出 `88`,  `32` 和 `56`的和.

如果在 [body_filter_by_lua](http://wiki.nginx.org/HttpLuaModule#body_filter_by_lua) 或 [body_filter_by_lua_file](http://wiki.nginx.org/HttpLuaModule#body_filter_by_lua_file)中使用这个table, 第一个元素保存的是发给输出过滤器的数据块，第二个元素保存的是标记整个输出数据流"eof"的boolean值。

发往下游Nginx输出过滤器的数据块和 "eof" 标记可以通过对表中相应元素的直接赋值来进行修改。如果将`ngx.arg[1]`设置成 `nil` 或空的Lua string , 将不会有数据发往下游 Nginx输出过滤器。

ngx.var.VARIABLE
----------------
**语法:** *ngx.var.VAR_NAME*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua**

读写Nginx变量的值.


    value = ngx.var.some_nginx_variable_name
    ngx.var.some_nginx_variable_name = value


注意只有已经定义了的nginx变量可以被写入.
例如:


    location /foo {
        set $my_var ''; # 这一行是必需的，在config期间创建 $my_var 
        content_by_lua '
            ngx.var.my_var = 123;
            ...
        ';
    }


也就是说，nginx变量不能随需创建.

有些nginx变量，如 `$args` 和 `$limit_rate` 可以被设值,
有些则不行, 如 `$arg_PARAMETER`.

Nginx 正则表达式组捕捉变量 `$1`, `$2`, `$3`等也可以使用这一接口来读取, 写法是 `ngx.var[1]`, `ngx.var[2]`, `ngx.var[3]`, 等等.

将 `ngx.var.Foo` 设置为 `nil` 将取消 `$Foo` Nginx 变量的定义. 


    ngx.var.args = nil


**警告** 在读取Nginx变量时, Nginx将在每个请求的内存池中分配内存，只有当请求结束时，这个内存池才被释放。所以如果在Lua代码中需要多次读取一个Nginx变量, 请将Nginx变量的值赋给你自己的Lua变量, 例如,


    local val = ngx.var.some_var
    --- 后续重复使用val


以避免在当前请求生命周期中出现（临时的）内存泄漏.

核心常量
------------
**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, *log_by_lua*, ngx.timer.**


      ngx.OK (0)
      ngx.ERROR (-1)
      ngx.AGAIN (-2)
      ngx.DONE (-4)
      ngx.DECLINED (-5)


注意这些常量中只有三个能被 [可供Lua调用的Nginx API](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)使用 ([ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit) 接受 `NGX_OK`, `NGX_ERROR`, and `NGX_DECLINED` 作为输入).


      ngx.null


`ngx.null` 常量是一个 `NULL` 轻量 userdata，通常用于表示Lua table等中的nil值，类似于 [lua-cjson](http://www.kyne.com.au/~mark/software/lua-cjson.php) 库的`cjson.null` 常量. 此常量在 `v0.5.0rc5` 版本中引入.

`ngx.DECLINED` 常量在 `v0.5.0rc19` 版本中引入.

HTTP 请求方法常量
-----------------------
**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**


      ngx.HTTP_GET
      ngx.HTTP_HEAD
      ngx.HTTP_PUT
      ngx.HTTP_POST
      ngx.HTTP_DELETE
      ngx.HTTP_OPTIONS   ( v0.5.0rc24 版本引入)
      ngx.HTTP_MKCOL     ( v0.8.2 版本引入)
      ngx.HTTP_COPY      ( v0.8.2 版本引入)
      ngx.HTTP_MOVE      ( v0.8.2 版本引入)
      ngx.HTTP_PROPFIND  ( v0.8.2 版本引入)
      ngx.HTTP_PROPPATCH ( v0.8.2 版本引入)
      ngx.HTTP_LOCK      ( v0.8.2 版本引入)
      ngx.HTTP_UNLOCK    ( v0.8.2 版本引入)
      ngx.HTTP_PATCH     ( v0.8.2 版本引入)
      ngx.HTTP_TRACE     ( v0.8.2 版本引入)

这些常量多用于 [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) and [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi) 调用.

HTTP status constants
---------------------
**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**


      value = ngx.HTTP_OK (200)
      value = ngx.HTTP_CREATED (201)
      value = ngx.HTTP_SPECIAL_RESPONSE (300)
      value = ngx.HTTP_MOVED_PERMANENTLY (301)
      value = ngx.HTTP_MOVED_TEMPORARILY (302)
      value = ngx.HTTP_SEE_OTHER (303)
      value = ngx.HTTP_NOT_MODIFIED (304)
      value = ngx.HTTP_BAD_REQUEST (400)
      value = ngx.HTTP_UNAUTHORIZED (401)
      value = ngx.HTTP_FORBIDDEN (403)
      value = ngx.HTTP_NOT_FOUND (404)
      value = ngx.HTTP_NOT_ALLOWED (405)
      value = ngx.HTTP_GONE (410)
      value = ngx.HTTP_INTERNAL_SERVER_ERROR (500)
      value = ngx.HTTP_METHOD_NOT_IMPLEMENTED (501)
      value = ngx.HTTP_SERVICE_UNAVAILABLE (503)
      value = ngx.HTTP_GATEWAY_TIMEOUT (504) (first added in the v0.3.1rc38 release)


Nginx 日志级别常量
------------------------
**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**


      ngx.STDERR
      ngx.EMERG
      ngx.ALERT
      ngx.CRIT
      ngx.ERR
      ngx.WARN
      ngx.NOTICE
      ngx.INFO
      ngx.DEBUG


这些常量通常被用在 [ngx.log](http://wiki.nginx.org/HttpLuaModule#ngx.log) 方法中.

print
-----
**语法:** *print(...)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**

将参数值以`ngx.NOTICE`级别写入nginx `error.log` 文件.

等价于


    ngx.log(ngx.NOTICE, ...)


可接受Lua `nil` 参数，写入的值为 `"nil"` 字符串， 而Lua boolean值会被写入成 `"true"` 或 `"false"` 字符串. `ngx.null` 常量也写入为 `"null"` 字符串.

Nginx 内核中硬编码限制了错误信息长度为 `2048` 字节. 这个限制包括了最后的换行和头部的时间戳。如果消息长度超限，Nginx 会做相应的截断。可以手工修改Nginx代码树中的`src/core/ngx_log.h`文件中的 `NGX_MAX_ERROR_STR` 宏定义来修改这个上限。

ngx.ctx
-------
**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**

这个table可用于存储每个请求的Lua上下文数据，其生命周期与当前请求一致 (和 Nginx 变量一样). 

考虑以下示例,


    location /test {
        rewrite_by_lua '
            ngx.say("foo = ", ngx.ctx.foo)
            ngx.ctx.foo = 76
        ';
        access_by_lua '
            ngx.ctx.foo = ngx.ctx.foo + 3
        ';
        content_by_lua '
            ngx.say(ngx.ctx.foo)
        ';
    }


`GET /test` 的输出如下


    foo = nil
    79


这说明 `ngx.ctx.foo` 在请求的rewrite, access, 和 content各个阶段都有效。

每个请求, 包括子请求, 都有对这个table的独立拷贝。例如:


    location /sub {
        content_by_lua '
            ngx.say("sub pre: ", ngx.ctx.blah)
            ngx.ctx.blah = 32
            ngx.say("sub post: ", ngx.ctx.blah)
        ';
    }
 
    location /main {
        content_by_lua '
            ngx.ctx.blah = 73
            ngx.say("main pre: ", ngx.ctx.blah)
            local res = ngx.location.capture("/sub")
            ngx.print(res.body)
            ngx.say("main post: ", ngx.ctx.blah)
        ';
    }


`GET /main` 将输出


    main pre: 73
    sub pre: nil
    sub post: 32
    main post: 73


这里，对子请求中的 `ngx.ctx.blah` 进行修改不会影响其父请求中的值. 这是因为它们是两个不同的版本.

内部重定向将删除原始请求的 request `ngx.ctx` 数据(如果有的话)，新的请求的 `ngx.ctx` 是空table. 例如,


    location /new {
        content_by_lua '
            ngx.say(ngx.ctx.foo)
        ';
    }
 
    location /orig {
        content_by_lua '
            ngx.ctx.foo = "hello"
            ngx.exec("/new")
        ';
    }


`GET /orig` 的结果


    nil


而不是原始的 `"hello"` 值.

任何数据，包括Lua closure和嵌套的table，都可以被放入这个“神奇的”table. 而且还允许注册自定义的元方法。

用一个新的Lua table来覆盖 `ngx.ctx` 也是支持的, 例如,


    ngx.ctx = { foo = 32, bar = 54 }


ngx.location.capture
--------------------
**语法:** *res = ngx.location.capture(uri, options?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

对`uri`发起一个同步的但仍是非阻塞的 *Nginx子请求*。 

Nginx子请求提供了向其它location发起非阻塞的内部请求的强大手段，这些location可以用磁盘文件路径或 *任何* 其它ngixn C 模块如 `ngx_proxy`, `ngx_fastcgi`, `ngx_memc`,
`ngx_postgres`, `ngx_drizzle`, 甚至是ngx_lua自己等等来配置.

请注意子请求只是模拟了 HTTP 接口，但事实上 *并没有* 额外的 HTTP/TCP 流量 *也没有* IPC 调用. 一切都在内部C这一层高效运行。

子请求与 HTTP 301/302 重定向(通过 [ngx.redirect](http://wiki.nginx.org/HttpLuaModule#ngx.redirect)) 和 内部重定向 (通过[ngx.exec](http://wiki.nginx.org/HttpLuaModule#ngx.exec)) 是完全不同的.

这是一个基本的例子:


    res = ngx.location.capture(uri)


返回一个Lua table，内容包括三项 (`res.status`, `res.header`, `res.body`, and `res.truncated`).

`res.status` 的值是子请求响应的响应状态码。

`res.header` 的值是子请求的所有响应头，这是一个正常的Lua table. 对于有多值的响应头,
其值是装有按出现顺序排序的所有值的 Lua (数组) table例如， 如果子请求响应头包含 :


    Set-Cookie: a=3
    Set-Cookie: foo=bar
    Set-Cookie: baz=blah


那么 `res.header["Set-Cookie"]` 的值是table
`{"a=3", "foo=bar", "baz=blah"}`.

`res.body` 保存子请求响应体数据，有可能被截断。你总是需要检查 `res.truncated` boolean 标记来确认 `res.body` 是否包含被截断的数据.

URI query string可以被添加到URI本身 如,


    res = ngx.location.capture('/foo/bar?a=3&b=4')


由于nginx内核的限制， 不允许有`@foo` 这样的具名location。对普通的location使用 `internal` 命令来准备只作内部使用的location.

可以为第2个可选参数提供一个选项table, 支持如下选项:

* `method`
	指定子请求的请求方法，只接受 `ngx.HTTP_POST` 这样的常量.
* `body`
	指定子请求的请求体 (只允许字符串值).
* `args`
	指定子请求的URI请求参数 (接受字符串值和Lua table)
* `ctx`
	为子请求指定 [ngx.ctx](http://wiki.nginx.org/HttpLuaModule#ngx.ctx) table. 这个值可以是当前请求的 [ngx.ctx](http://wiki.nginx.org/HttpLuaModule#ngx.ctx) table, 其结果是父请求和子请求共享同一个context able本命令在 `v0.3.1rc25` 版本中引入.
* `vars`
	Lua table，其内部的值将作为子请求的Nginx 变量本命令在 `v0.3.1rc31` 版本中引入.
* `copy_all_vars`
	指定是否将当前请求中的所有Ngin变量复制到子请求中在子请求中修改nginx变量不会影响当前（父）请求. 本命令在 `v0.3.1rc31` 版本中引入.
* `share_all_vars`
	指定是否将本（父）请求中的Nginx变量值与子请求共享。在子请求中修改nginx变量将影响到当前（父）请求.

例如，可以这样发起一个POST子请求


    res = ngx.location.capture(
        '/foo/bar',
        { method = ngx.HTTP_POST, body = 'hello, world' }
    )


除POST外还有其它的HTTP方法常量。默认的 `method` 选项是 `ngx.HTTP_GET` .

 <code>args</code> 选项可以用来指定其它的 URI 参数, 例如,


    ngx.location.capture('/foo?a=1',
        { args = { b = 3, c = ':' } }
    )


等价于


    ngx.location.capture('/foo?a=1&b=3&c=%3a')


也就是说, 这个方法会根据 URI 规范对参数的key和value进行转义并将其拼接成一个完整的 query string.  `args` 参数Lua table的格式与 [ngx.encode_args](http://wiki.nginx.org/HttpLuaModule#ngx.encode_args) 所使用的table格式一样.

`args` 选项也可以直接使用 query string字符串:


    ngx.location.capture('/foo?a=1',
        { args = 'b=3&c=%3a' } }
    )


这段代码与前面的代码在功能上完全一致。

`share_all_vars`选项控制是否将本请求中的Nginx变量值与子请求共享。如果这个选项被置为 `true`, 则当前的请求与相关的子请求会共同同样的Nginx 变量域. 这样，子请求中对Nginx变量的修改将会影响到当前请求.

要谨慎使用这个选项，因为变量域的共享可能会带来预料之外的副作用。更推荐使用 `args`, `vars`, 或 `copy_all_vars` 这些选项.

此选项默认设为 `false` 


    location /other {
        set $dog "$dog world";
        echo "$uri dog: $dog";
    }

    location /lua {
        set $dog 'hello';
        content_by_lua '
            res = ngx.location.capture("/other",
                { share_all_vars = true });

            ngx.print(res.body)
            ngx.say(ngx.var.uri, ": ", ngx.var.dog)
        ';
    }


Accessing location `/lua` gives


    /other dog: hello world
    /lua: hello world


 <code>copy_all_vars</code> 在发起子请求时为子请求提供父请求Nginx变量的拷贝。子请求对这些变量的修改不会影响到父请求和那些分享父请求变量的子请求。.


    location /other {
        set $dog "$dog world";
        echo "$uri dog: $dog";
    }

    location /lua {
        set $dog 'hello';
        content_by_lua '
            res = ngx.location.capture("/other",
                { copy_all_vars = true });

            ngx.print(res.body)
            ngx.say(ngx.var.uri, ": ", ngx.var.dog)
        ';
    }


`GET /lua` 的结果是


    /other dog: hello world
    /lua: hello


注意，如果 `share_all_vars` 和 `copy_all_vars` 都设为true, `share_all_vars` 优先.

除了上面两种设置外，也可以在子请求中通过 `vars` 选项设置变量. 这些变量的设置发生在共享或拷贝的变量被求值以后， 这样提供了一种比将参数编码成URL参数并在Nginx配置文件中对它们进行反转义更高效的传递参数的方法。


    location /other {
        content_by_lua '
            ngx.say("dog = ", ngx.var.dog)
            ngx.say("cat = ", ngx.var.cat)
        ';
    }

    location /lua {
        set $dog '';
        set $cat '';
        content_by_lua '
            res = ngx.location.capture("/other",
                { vars = { dog = "hello", cat = 32 }});

            ngx.print(res.body)
        ';
    }


访问 `/lua` 的结果


    dog = hello
    cat = 32


 <code>ctx</code> 选项用来为子请求指定自定义的 [ngx.ctx](http://wiki.nginx.org/HttpLuaModule#ngx.ctx) table 。


    location /sub {
        content_by_lua '
            ngx.ctx.foo = "bar";
        ';
    }
    location /lua {
        content_by_lua '
            local ctx = {}
            res = ngx.location.capture("/sub", { ctx = ctx })

            ngx.say(ctx.foo);
            ngx.say(ngx.ctx.foo);
        ';
    }


请求 `GET /lua` 得到


    bar
    nil


也可以用 `ctx` 在当前（父）请求和子请求之间共享同一个 [ngx.ctx](http://wiki.nginx.org/HttpLuaModule#ngx.ctx) table:


    location /sub {
        content_by_lua '
            ngx.ctx.foo = "bar";
        ';
    }
    location /lua {
        content_by_lua '
            res = ngx.location.capture("/sub", { ctx = ngx.ctx })
            ngx.say(ngx.ctx.foo);
        ';
    }


请求 `GET /lua` 的结果


    bar


注意在默认情况下 [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) 发起的子请求将从当前请求中继承所有的请求头，这可能会对子请求的响应带来意料之外的副作用。例如，如果用标准的 `ngx_proxy` 模块来为子请求服务, 主请求中的 "Accept-Encoding: gzip" 头可能会导致无法被Lua代码正常处理的被压缩的响应。在子请求的location中应设置 
[proxy_pass_request_headers](http://wiki.nginx.org/HttpProxyModule#proxy_pass_request_headers) 为 `off`来忽略主请求的头.

如果没有指定 `body` 选项， `POST` 和 `PUT` 子请求将从父请求中继承请求体(如果有的话).

对每个主请求，对同时并发的子请求的数量有一个硬编码的上限。在旧版本的Nginx中, 这个上限是 `50` 而在最近的版本, Nginx `1.1.x` 以上, 这个上限增加到了 `200`。如果超限， `error.log` 文件中将记录以下错误日志:


    [error] 13983#0: *1 subrequests cycle while processing "/uri"


可以手工修改Nginx代码树中的`nginx/src/http/ngx_http_request.h`文件中的 `NGX_HTTP_MAX_SUBREQUESTS` 宏定义来修改这个上限。

请参阅 [其它模块的子请求命令](http://wiki.nginx.org/HttpLuaModule#Locations_Configured_by_Subrequest_Directives_of_Other_Modules)了解子请求location的限制.

ngx.location.capture_multi
--------------------------
**语法:** *res1, res2, ... = ngx.location.capture_multi({ {uri, options?}, {uri, options?}, ... })*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

同[ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture), 但是支持并行的多个子请求.

这个函数发起在输入table中指定的若干并行子请求，并以指定的顺序返回其结果。例如,


    res1, res2, res3 = ngx.location.capture_multi{
        { "/foo", { args = "a=3&b=4" } },
        { "/bar" },
        { "/baz", { method = ngx.HTTP_POST, body = "hello" } },
    }
 
    if res1.status == ngx.HTTP_OK then
        ...
    end
 
    if res2.body == "BLAH" then
        ...
    end


这个函数只有在所有子请求都结束后才会返回。总的延迟是各个子请求延时的最大值，而不是它们的和.

如果要发起的子请求的数量事先并不确定，可以在请求或响应中使用Lua table:


    -- 构造请求table
    local reqs = {}
    table.insert(reqs, { "/mysql" })
    table.insert(reqs, { "/postgres" })
    table.insert(reqs, { "/redis" })
    table.insert(reqs, { "/memcached" })
 
    -- 一次性发起所有请求，等待所有请求都返回
    local resps = { ngx.location.capture_multi(reqs) }
 
    -- 对响应 table进行遍历
    for i, resp in ipairs(resps) do
        -- 对响应table "resp"进行处理
    end


 [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) 函数只是本函数的特殊形式从逻辑上说, [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) 可以这样来实现


    ngx.location.capture =
        function (uri, args)
            return ngx.location.capture_multi({ {uri, args} })
        end


请参阅 [其它模块的子请求命令](http://wiki.nginx.org/HttpLuaModule#Locations_Configured_by_Subrequest_Directives_of_Other_Modules)了解子请求location的限制.

ngx.status
----------
**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

读写当前请求的响应状态码。对它的调用应在发送响应头之前。


    ngx.status = ngx.HTTP_CREATED
    status = ngx.status


在发送了响应头之后设置 `ngx.status` 没有作用，但会在ngxin错误日志文件里产生一条错误信息: 


    attempt to set ngx.status after sending out response headers


ngx.header.HEADER
-----------------
**语法:** *ngx.header.HEADER = VALUE*

**语法:** *value = ngx.header.HEADER*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

对当前请求的 `HEADER` 响应头进行设置、添加和清除.

默认会将头名称中的下划线(`_`) 换成减号 (`-`) . 这个转换可以通过 [lua_transform_underscores_in_response_headers](http://wiki.nginx.org/HttpLuaModule#lua_transform_underscores_in_response_headers) 命令来关闭.

头名称的匹配是大小写无关的.


    -- 等价于 ngx.header["Content-Type"] = 'text/plain'
    ngx.header.content_type = 'text/plain';
 
    ngx.header["X-My-Header"] = 'blah blah';


多值头可以这样设置:


    ngx.header['Set-Cookie'] = {'a=32; path=/', 'b=4; path=/'}


将会在响应头中生成


    Set-Cookie: a=32; path=/
    Set-Cookie: b=4; path=/


. 

仅支持 Lua table (对于只支持单个值的标准头如`Content-Type`，只有table中的最后一个元素起作用


    ngx.header.content_type = {'a', 'b'}


等价于


    ngx.header.content_type = 'b'


将值设置成 `nil` 的结果是将它从响应头中删除:


    ngx.header["X-My-Header"] = nil;


设置成一个空table的效果是一样的:


    ngx.header["X-My-Header"] = {};


在发送完响应头 (用 [[#ngx.send_headersngx.send_headers]]显式发送 或 用 [[#ngx.printngx.print]] 和类似方法隐式发送)设置`ngx.header.HEADER` 将抛出Lua异常.

读取 `ngx.header.HEADER` 将返回名为 `HEADER`的响应头. 

同样，头名称中的下划线 (`_`) 将被替换成减号 (`-`) ，头名称的匹配是大小写无关的。如果没有响应头，则返回 `nil`.

这在 [header_filter_by_lua](http://wiki.nginx.org/HttpLuaModule#header_filter_by_lua) 和 [header_filter_by_lua_file](http://wiki.nginx.org/HttpLuaModule#header_filter_by_lua_file)的上下文中特别有用, 例如,


    location /test {
        set $footer '';

        proxy_pass http://some-backend;

        header_filter_by_lua '
            if ngx.header["X-My-Header"] == "blah" then
                ngx.var.footer = "some value"
            end
        ';

        echo_after_body $footer;
    }


对于多值头，所有的值将按顺序放在一个Lua table中返回. 例如，响应头


    Foo: bar
    Foo: baz


的结果是在读取`ngx.header.Foo`时返回


    {"bar", "baz"}


.

注意 `ngx.header` 不是一个正常的Lua table，所以不能使用`ipairs`函数来对它进行遍历.

要读取 *请求* 头, 用 [ngx.req.get_headers](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_headers) 函数.

ngx.req.start_time
------------------
**语法:** *secs = ngx.req.start_time()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua**

返回表示当前请求创建时间的浮点数 (小数部分为毫秒).

下面的例子用纯Lua代码模拟 `$request_time` 变量的值 (由 [HttpLogModule](http://wiki.nginx.org/HttpLogModule) 提供) in pure Lua:


    local request_time = ngx.now() - ngx.req.start_time()


本命令在 `v0.7.7` 版本中引入.

参阅 [ngx.now](http://wiki.nginx.org/HttpLuaModule#ngx.now) 和 [ngx.update_time](http://wiki.nginx.org/HttpLuaModule#ngx.update_time).

ngx.req.http_version
--------------------
**语法:** *num = ngx.req.http_version()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

以Lua number的形式返回当前请求的HTTP版本号.

目前可能的值有 1.0, 1.1, 和 0.9. 遇到不支持的版本号返回`nil`.

本命令在 `v0.7.17` 版本中引入.

ngx.req.raw_header
------------------
**语法:** *str = ngx.req.raw_header(no_request_line?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

返回Nginx服务器接收到的原始HTTP协议头.

默认情况下，包含请求行及结尾的 `CR LF`. 例如,


    ngx.print(ngx.req.raw_header())


输出象这样:


    GET /t HTTP/1.1
    Host: localhost
    Connection: close
    Foo: bar



要从结果中去掉请求行，可以将可选的
`no_request_line` 参数设为 `true` . 例如,


    ngx.print(ngx.req.raw_header(true))


输出象这样:


    Host: localhost
    Connection: close
    Foo: bar



本命令在 `v0.7.17` 版本中引入.

ngx.req.get_method
------------------
**语法:** *method_name = ngx.req.get_method()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

获得当前请求的请求方法名称. 返回值是如`"GET"` 和 `"POST"` 这样的字符串而不是数字 [方法常量](http://wiki.nginx.org/HttpLuaModule#HTTP_method_constants).

如果当前请求是Nginx子请求, 将返回子请求的请求方法名称.

本命令在 `v0.5.6` 版本中引入.

参阅[ngx.req.set_method](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_method).

ngx.req.set_method
------------------
**语法:** *ngx.req.set_method(method_id)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

用`request_id`参数覆盖当前请求的请求方法. 当前仅支持数字 [方法常量](http://wiki.nginx.org/HttpLuaModule#HTTP_method_constants), 如`ngx.HTTP_POST` 和 `ngx.HTTP_GET`.

如果当前请求是Nginx子请求, 则子请求的请求方法将被设置为新值.

本命令在 `v0.5.6` 版本中引入.

参阅 [ngx.req.get_method](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_method).

ngx.req.set_uri
---------------
**语法:** *ngx.req.set_uri(uri, jump?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua**

用`uri`参数覆盖当前请求的 (解析过的) URI.  `uri` 参数必须是一个Lua string，长度必须大于0，不然会抛出Lua异常.

可选的boolean参数 `jump` 可以象 [HttpRewriteModule](http://wiki.nginx.org/HttpRewriteModule) 的 [[HttpRewriteModule#rewriterewrite]] 命令一样触发location重匹配 (或称location跳转) , 如果 `jump` 是 `true` (默认为`false`), 这个函数将不会返回，而是告诉 Nginx 在后面的`post-rewrite`阶段尝试重新对新的URI值进行location匹配并中转到新的location.

否则不会触发Location跳转, 而只是改变当前请求的 URI, 这也是默认的行为. `jump`参数为`false`或不提供时，这个函数会返回，但没有返回值。

例如，下面的nginx配置代码


    rewrite ^ /foo last;


可以用Lua这样写:


    ngx.req.set_uri("/foo", true)


类似的, Nginx 配置


    rewrite ^ /foo break;


可以用Lua写成


    ngx.req.set_uri("/foo", false)


或等价的,


    ngx.req.set_uri("/foo")


 <code>jump</code> 只能在[[#rewrite_by_luarewrite_by_lua]] 和 [[#rewrite_by_lua_filerewrite_by_lua_file]]中被设为 <code>true</code>. 在其它的上下文中使用jump是被禁止的，会抛出Lua异常。

下面是一个更复杂的包含正则替换的例子


    location /test {
        rewrite_by_lua '
            local uri = ngx.re.sub(ngx.var.uri, "^/test/(.*)", "$1", "o")
            ngx.req.set_uri(uri)
        ';
        proxy_pass http://my_backend;
    }


在功能上等价于


    location /test {
        rewrite ^/test/(.*) /$1 break;
        proxy_pass http://my_backend;
    }


注意不能使用这个接口来重写URI参数，请使用 [ngx.req.set_uri_args](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_uri_args). 例如，Nginx配置


    rewrite ^ /foo?a=3? last;


可以写成


    ngx.req.set_uri_args("a=3")
    ngx.req.set_uri("/foo", true)


或


    ngx.req.set_uri_args({a = 3})
    ngx.req.set_uri("/foo", true)


本命令在 `v0.3.1rc14` 版本中引入.

ngx.req.set_uri_args
--------------------
**语法:** *ngx.req.set_uri_args(args)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua**

用`args`覆盖当前请求的URI请求参数.  `args` 参数可以是 Lua 字符串, 如


    ngx.req.set_uri_args("a=3&b=hello%20world")


或存有请求参数名－值对的 Lua table, 如


    ngx.req.set_uri_args({ a = 3, b = "hello world" })


对后一种情况, 本方法会用URI转义规则对key和value进行转义.

也支持多值参数:


    ngx.req.set_uri_args({ a = 3, b = {5, 6} })


生成的query string是 `a=3&b=5&b=6`.

本命令在 `v0.3.1rc13` 版本中引入.

参阅[ngx.req.set_uri](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_uri).

ngx.req.get_uri_args
--------------------
**语法:** *args = ngx.req.get_uri_args(max_args?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

返回存有当前请求URL请求参数的Lua table.


    location = /test {
        content_by_lua '
            local args = ngx.req.get_uri_args()
            for key, val in pairs(args) do
                if type(val) == "table" then
                    ngx.say(key, ": ", table.concat(val, ", "))
                else
                    ngx.say(key, ": ", val)
                end
            end
        ';
    }


`GET /test?foo=bar&bar=baz&bar=blah` 的结果是响应体


    foo: bar
    bar: baz, blah


如果一个参数的key出现多次，则其value是一个按出现顺序保存多个值的table

key和value都根据URI转义规则进行转义. 在上面的设置中, `GET /test?a%20b=1%61+2` 的结果为:


    a b: 1a 2


没有 `=<value>` 的部分将被视为 boolean 参数. `GET /test?foo&bar` 的结果:


    foo: true
    bar: true


即他们的值为 boolean 值`true`. 要注意，这与value为空字符串的参数是不一样的. `GET /test?foo=&bar=` will give something like


    foo: 
    bar: 


空的key参数将被丢弃. 例如，`GET /test?=hello&=world` 的结果为空.

还支持在运行时用ngxin变量 `$args` (或Lua `ngx.var.args` )来更新请求参数:


    ngx.var.args = "a=3&b=42"
    local args = ngx.req.get_uri_args()


这里 `args` table 的内容为


    {a = 3, b = 42}


，与实际的请求query string无关.

注意，默认最多只解析100个请求参数 (包括名称一样的) ，多出来的请求参数将被简单丢弃，以防止DOS攻击.

但是, 可以用可选的`max_args` 函数参数修改这个上限:


    local args = ngx.req.get_uri_args(10)


这个参数设成0表示无上限，会处理接收到的所有的请求参数：


    local args = ngx.req.get_uri_args(0)


强烈不建议去掉`max_args` 限制.

ngx.req.get_post_args
---------------------
**语法:** *args, err = ngx.req.get_post_args(max_args?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

返回当前所有请求的POST请求参数 (MIME类型为 `application/x-www-form-urlencoded`). 为避免出错，先调用 [ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body) 读取请求体或打开 [lua_need_request_body](http://wiki.nginx.org/HttpLuaModule#lua_need_request_body) 命令.


    location = /test {
        content_by_lua '
            ngx.req.read_body()
            local args = ngx.req.get_post_args()
            if not args then
                ngx.say("failed to get post args: ", err)
                return
            end
            for key, val in pairs(args) do
                if type(val) == "table" then
                    ngx.say(key, ": ", table.concat(val, ", "))
                else
                    ngx.say(key, ": ", val)
                end
            end
        ';
    }


Then


    # Post请求体为 'foo=bar&bar=baz&bar=blah'
    $ curl --data 'foo=bar&bar=baz&bar=blah' localhost/test


输出


    foo: bar
    bar: baz, blah


如果一个参数的键出现多次，则生成一个包含有所有参数值的table，table中的元素次序与值出现的次序相同.

键和值都根据URI转义规则进行转义.

按以上设置,


    # POST 体为 'a%20b=1%61+2'
    $ curl -d 'a%20b=1%61+2' localhost/test


将输出:


    a b: 1a 2


没有 `=<value>` 的部分将被视为 boolean 参数. `GET /test?foo&bar` 的结果:


    foo: true
    bar: true


即他们的值为 boolean 值`true`. 要注意，这与value为空字符串的参数是不一样的. 以请求体`foo=&bar=`发送 `POST /test`请求将返回


    foo: 
    bar: 


空的key参数将被丢弃. 例如，以请求体`=hello&=world`发送`POST /test` 将获得空白输出.

注意，默认最多只解析100个请求参数 (包括名称一样的) ，多出来的请求参数将被简单丢弃，以防止DOS攻击.  

但是, 可以用可选的`max_args` 函数参数修改这个上限:


    local args = ngx.req.get_post_args(10)


这个参数设成0表示无上限，会处理接收到的所有的请求参数：


    local args = ngx.req.get_post_args(0)


强烈不建议去掉`max_args` 限制.

ngx.req.get_headers
-------------------
**syntax:** *headers = ngx.req.get_headers(max_headers?, raw?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

返回持有当前请求所有头的Lua table.


    local h = ngx.req.get_headers()
    for k, v in pairs(h) do
        ...
    end


要读取某一个头:


    ngx.say("Host: ", ngx.req.get_headers()["Host"])


注意，使用内核模块[[HttpCoreModule#$http_HEADER$http_HEADER]]的 [ngx.var.HEADER](http://wiki.nginx.org/HttpLuaModule#ngx.var.VARIABLE) API, 可能更适合用来读取某一个头信息。

对于有多个值的请求头如:


    Foo: foo
    Foo: bar
    Foo: baz


`ngx.req.get_headers()["Foo"]` 的值是这样的 Lua (数组) table:


    {"foo", "bar", "baz"}


注意，默认最多只解析100个请求头 (包括名称一样的) ，多出来的请求头将被简单丢弃，以防止DOS攻击.  

但是, 可以用可选的`max_headers` 函数参数修改这个上限:


    local args = ngx.req.get_headers(10)


这个参数设成0表示无上限，会处理接收到的所有的请求头：


    local args = ngx.req.get_headers(0)


去除`max_headers` 限制是非常不推荐的.

从`0.6.9`版本开始, 默认Lua table中的所有头名称将被转换成纯小写，除非 `raw` 参数被设为 `true` (默认为`false`).

同样, 默认会为结果Lua table添加一个 `__index` 元方法，它会将所有的key都转换成纯小写形式，并且将所有的下划线转换成减号，以防止查找失败。例如，如果有一个请求头 `My-Foo-Header` , 以下各种写法都能正确获得这个头的值:


    ngx.say(headers.my_foo_header)
    ngx.say(headers["My-Foo-Header"])
    ngx.say(headers["my-foo-header"])


如果`raw`参数被设为`true`，则不会添加 `__index` 元方法。

ngx.req.set_header
------------------
**语法:** *ngx.req.set_header(header_name, header_value)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*

将当前请求名为 `header_name` 的头的值设为`header_value`, 替换其当前的值.

所有用[[#ngx.location.capturengx.location.capture]] and [[#ngx.location.capture_multingx.location.capture_multi]]发起的子请求都默认继承新设置的头。

以下是设置 `Content-Length` 头的例子:


    ngx.req.set_header("Content-Type", "text/css")


 <code>header_value</code> 可以接受一个数组存放多个值,
例如,


    ngx.req.set_header("Foo", {"a", "abc"})


将生成两个请求头:


    Foo: a
    Foo: abc


如果之前设置了 `Foo` 头，旧的值将被覆盖.

如果 `header_value` 参数是`nil`, 这个请求头将被删除. 所以


    ngx.req.set_header("X-Foo", nil)


等价于


    ngx.req.clear_header("X-Foo")


ngx.req.clear_header
--------------------
**语法:** *ngx.req.clear_header(header_name)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua**

清除当前请求中名为 `header_name` 的头. 当前请求的任何子请求都不会受影响.

ngx.req.read_body
-----------------
**语法:** *ngx.req.read_body()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

同步读取客户端请求体，但是不阻塞Nginx事件循环.


    ngx.req.read_body()
    local args = ngx.req.get_post_args()


如果打开了[[#lua_need_request_bodylua_need_request_body]]或其它模块导致请求体已被读取，则此函数不会执行，立即返回.

如果用[[#ngx.req.discard_bodyngx.req.discard_body]]函数或其它模块显式地丢弃了请求体，则此函数不会执行，立即返回.

在出错的情况下，如在读取数据时的连接错误, 此函数会抛出Lua异常 *或* 直接终止当前请求并立即返回 500 状态码.

使用此函数读取的请求体数据也可以延后，通过 [ngx.req.get_body_data](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_body_data) 读取，或使用缓存的磁盘的临时文件名[[#ngx.req.get_body_filengx.req.get_body_file]]进行读取. 这取决于

1. 当前请求体是否已经超过了 [client_body_buffer_size](http://wiki.nginx.org/HttpCoreModule#client_body_buffer_size) 的大小,
1. 是否打开了 [client_body_in_file_only](http://wiki.nginx.org/HttpCoreModule#client_body_in_file_only) .

有时候当前请求可能有但不一定有请求体，需要使用 [ngx.req.discard_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.discard_body) 函数来显式地丢弃请求体，以防止在HTTP 1.1 keepalive或HTTP 1.1 pipelining时出错。

本命令在 `v0.3.1rc17` 版本中引入.

ngx.req.discard_body
--------------------
**语法:** *ngx.req.discard_body()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

显式地丢弃请求体，即, 从连接中读取数据但是马上丢掉它. 请注意，无视请求体（译注：无视指认为请求体不存在，与本方法的不同是不会从连接中读取请求体数据）不是丢弃它的正确方法, 因此必须调用本方法来防止在HTTP 1.1 keepalive 或 HTTP 1.1 pipelining时出错.

本函数是异步调用，会立即返回.

如果请求体已经被读取过了，则本函数不做任何事，直接返回.

本命令在 `v0.3.1rc17` 版本中引入.

参阅[ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body).

ngx.req.get_body_data
---------------------
**语法:** *data = ngx.req.get_body_data()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

获取内存中的请求体数据. 它返回装有所有解析过的请求参数的Lua字符串，而不是Lua table. 如果需要Lua table请使用[ngx.req.get_post_args](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_post_args) 函数.

在以下情况下此函数返回 `nil` 
1. 请求体尚未被读取,
1. 请求体已经被读进磁盘临时文件,
1. 请求体长度为0.

如果请求体尚未被读取, 先调用[ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body) (或打开[lua_need_request_body](http://wiki.nginx.org/HttpLuaModule#lua_need_request_body) 来强制本模块读取请求体. 虽然我们并不推荐这样做).

如果请求体已经被读进磁盘临时文件, 试试调用[ngx.req.get_body_file](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_body_file) 函数.

要强制使用内存请求体，将 [client_body_buffer_size](http://wiki.nginx.org/HttpCoreModule#client_body_buffer_size) 设置成与 [client_max_body_size](http://wiki.nginx.org/HttpCoreModule#client_max_body_size) 同样的值.

注意此方法比 `ngx.var.request_body` 或 `ngx.var.echo_request_body` 更高效，因为它可以节省一次动态内存分配和一次数据拷贝。

本命令在 `v0.3.1rc17` 版本中引入.

参阅 [ngx.req.get_body_file](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_body_file).

ngx.req.get_body_file
---------------------
**语法:** *file_name = ngx.req.get_body_file()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

获取文件中请求体数据的文件名. 如果请求体尚未被读取或已经被读入内存，将返回`nil`。

返回的文件是只读的，通常由Nginx的内存池来清理. 还要在Lua代码中对其进行修改，重命名或删除。

如果请求体尚未被读取, 先调用[ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body) (或打开[lua_need_request_body](http://wiki.nginx.org/HttpLuaModule#lua_need_request_body) 来强制本模块读取请求体. 虽然我们并不推荐这样做).

如果请求体已经被读入内存, 试试调用 [ngx.req.get_body_data](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_body_data) .

要强制使用文件请求体, 打开 [client_body_in_file_only](http://wiki.nginx.org/HttpCoreModule#client_body_in_file_only).

本命令在 `v0.3.1rc17` 版本中引入.

参阅[ngx.req.get_body_data](http://wiki.nginx.org/HttpLuaModule#ngx.req.get_body_data).

ngx.req.set_body_data
---------------------
**语法:** *ngx.req.set_body_data(data)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

将当前请求的请求体设置成`data` 参数指定的内存数据 .

如果当前请求的请求体尚未被读取，则其将被正确地丢弃. 如果当前请求的请求体已经被读入内存，则旧的请求体内存将被释放；如果已经被读入磁盘文件，则此文件将被立即删除。

本命令在 `v0.3.1rc18` 版本中引入.

参阅 [ngx.req.set_body_file](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_body_file).

ngx.req.set_body_file
---------------------
**语法:** *ngx.req.set_body_file(file_name, auto_clean?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

将当前请求的请求体设置成`file_name` 参数指定的文件的内容.

如果可选参数 `auto_clean` 被设置为 `true` , 则文件将在请求结束时，或同一次请求中再次调用 [ngx.req.set_body_data](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_body_data) 时被删除.  `auto_clean` 默认为 `false`.

请保证 `file_name` 参数指定的文件真实存在，并且正确设置其访问权限，便利Nginx worker进程能够访问它，以避免抛出Lua异常.

如果当前请求的请求体尚未被读取，则其将被正确地丢弃. 如果当前请求的请求体已经被读入内存，则旧的请求体内存将被释放；如果已经被读入磁盘文件，则此文件将被立即删除。

本命令在 `v0.3.1rc18` 版本中引入.

参阅 [ngx.req.set_body_data](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_body_data).

ngx.req.init_body
-----------------
**语法:** *ngx.req.init_body(buffer_size?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

为当前请求创建一个新的空请求体，初始化其缓冲区供后续的 [ngx.req.append_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.append_body) 和 [ngx.req.finish_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.finish_body) 写入.

如果指定了 `buffer_size` 参数, 则将会用这个值作为供 [[#ngx.req.append_bodyngx.req.append_body]] 写入的内存缓冲区的大小. 如果没有这个参数, 将使用标准 [client_body_buffer_size](http://wiki.nginx.org/HttpCoreModule#client_body_buffer_size) 命令指定的值.

当请求体数据不再能保留在内存中时，数据将被写到一个临时文件里，就象Nginx核心中的标准请求体读取器一样.

有一点很重要，在所有数据都已加到当前请求体中后，必须调用 [ngx.req.finish_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.finish_body) 。还有，如果此函数与 [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket) 一起使用, 必须在调用此函数之前 *先* 调用 [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket) , 否则将出现 "request body already exists" 错误.

这个函数的用法通常是这样的:


    ngx.req.init_body(128 * 1024)  -- 缓冲区大小128KB
    for chunk in next_data_chunk() do
        ngx.req.append_body(chunk) -- 每一块可以是 4KB
    end
    ngx.req.finish_body()


此函数与 [ngx.req.append_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.append_body), [ngx.req.finish_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.finish_body), 和 [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)一起使用，可以实现高效的纯Lua输入过滤器 (在[rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua)* 或 [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua)* 中), 过滤器可与其它 Nginx 内容处理器或上游模块 [HttpProxyModule](http://wiki.nginx.org/HttpProxyModule) 和 [HttpFastcgiModule](http://wiki.nginx.org/HttpFastcgiModule)一起使用.

本命令在 `v0.5.11` 版本中引入.

ngx.req.append_body
-------------------
**语法:** *ngx.req.append_body(data_chunk)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

将 `data_chunk` 参数指定的数据块加到用[[#ngx.req.init_bodyngx.req.init_body]]创建的请求体中。

当请求体数据不再能保留在内存中时，数据将被写到一个临时文件里，就象Nginx核心中的标准请求体读取器一样.

有一点很重要，在所有数据都已加到当前请求体中后，必须调用 [ngx.req.finish_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.finish_body) 。

此函数与 [ngx.req.iit_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.init_body), [ngx.req.finish_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.finish_body), 和 [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)一起使用，可以实现高效的纯Lua输入过滤器 (在[rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua)* 或 [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua)* 中), 过滤器可与其它 Nginx 内容处理器或上游模块 [HttpProxyModule](http://wiki.nginx.org/HttpProxyModule) 和 [HttpFastcgiModule](http://wiki.nginx.org/HttpFastcgiModule)一起使用.

本命令在 `v0.5.11` 版本中引入.

参阅 [ngx.req.init_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.init_body).

ngx.req.finish_body
-------------------
**语法:** *ngx.req.finish_body()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

完成由[[#ngx.req.init_bodyngx.req.init_body]] and [[#ngx.req.append_bodyngx.req.append_body]]实现的请求体创建过程。

此函数与 [ngx.req.iit_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.init_body), [ngx.req.append_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.append_body), 和 [ngx.req.socket](http://wiki.nginx.org/HttpLuaModule#ngx.req.socket)一起使用，可以实现高效的纯Lua输入过滤器 (在[rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua)* 或 [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua)* 中), 过滤器可与其它 Nginx 内容处理器或上游模块 [HttpProxyModule](http://wiki.nginx.org/HttpProxyModule) 和 [HttpFastcgiModule](http://wiki.nginx.org/HttpFastcgiModule)一起使用.

本命令在 `v0.5.11` 版本中引入.

参阅 [ngx.req.init_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.init_body).

ngx.req.socket
--------------
**语法:** *tcpsock, err = ngx.req.socket()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

返回包装了下游连接的只读cosocket对象. 此对象只支持 [receive](http://wiki.nginx.org/HttpLuaModule#tcpsock:receive) 和 [receiveuntil](http://wiki.nginx.org/HttpLuaModule#tcpsock:receiveuntil) 方法.

如果出错，将返回 `nil` 和 描述错误的字符串.

此方法返回的socket对象通常用来以流的方式读取当前请求的体。不要打开 [lua_need_request_body](http://wiki.nginx.org/HttpLuaModule#lua_need_request_body) 命令, 而且不要将本函数与 [ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body) 和 [ngx.req.discard_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.discard_body) 混用.

如果已经有请求体数据被预加载进了Nginx核心请求处理器缓冲区， 返回的cosocket对象将考虑到这一点，以避免这种预加载造成的潜在数据丢失。

本命令在 `v0.5.0rc1` 版本中引入.

ngx.exec
--------
**语法:** *ngx.exec(uri, args?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

使用`args`作为参数发起到 `uri` 的内部重定向 .


    ngx.exec('/some-location');
    ngx.exec('/some-location', 'a=3&b=5&c=6');
    ngx.exec('/some-location?a=3&b=5', 'c=6');


支持具名location，但是query string将被忽略. 例如,


    location /foo {
        content_by_lua '
            ngx.exec("@bar");
        ';
    }
 
    location @bar {
        ...
    }


可选的第二个参数 `args` 可用于指定额外的 URI 请求参数, 例如:


    ngx.exec("/foo", "a=3&b=hello%20world")


或者可以传一个Lua table给 `args` 参数，让ngx_lua去完成URI转义和字符串拼接.


    ngx.exec("/foo", { a = 3, b = "hello world" })


结果与上一个例子完全一样.  `args` 参数Lua table的格式与 [ngx.encode_args](http://wiki.nginx.org/HttpLuaModule#ngx.encode_args) 所使用的table格式一样.

注意，这跟 [ngx.redirect](http://wiki.nginx.org/HttpLuaModule#ngx.redirect) 是完全不同的，它只是一个内部重定向，并没有新的HTTP流量产生.

这个方法永不返回.

这个方法 *必须* 在 [ngx.send_headers](http://wiki.nginx.org/HttpLuaModule#ngx.send_headers) 或使用[[#ngx.printngx.print]] or [[#ngx.sayngx.say]]进行显式的响应体输出之前调用.

强烈建议将此函数与 `return` 一起使用, 即, `return ngx.exec(...)`.

此方法类似于 [HttpEchoModule](http://wiki.nginx.org/HttpEchoModule) 的 [echo_exec](http://wiki.nginx.org/HttpEchoModule#echo_exec) 命令.

ngx.redirect
------------
**语法:** *ngx.redirect(uri, status?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

发起到`uri`的 `HTTP 301` 或 `302` 重定向.

可选的 `status` 参数指定使用
`301` 还是 `302` 。默认值是 `302` (`ngx.HTTP_MOVED_TEMPORARILY`) .

以下的例子假设当前的服务器名为 `localhost` 监听 1984 端口:


    return ngx.redirect("/foo")


等价于


    return ngx.redirect("http://localhost:1984/foo", ngx.HTTP_MOVED_TEMPORARILY)


也支持重定向到任意的外部URLs , 例如:


    return ngx.redirect("http://www.google.com")


我们也可以直接在 `status` 参数中使用数字状态码:


    return ngx.redirect("/foo", 301)


这个方法 *必须* 在 [ngx.send_headers](http://wiki.nginx.org/HttpLuaModule#ngx.send_headers) 或使用[[#ngx.printngx.print]] or [[#ngx.sayngx.say]]进行显式的响应体输出之前调用.

此方法很象标准[HttpRewriteModule](http://wiki.nginx.org/HttpRewriteModule)中带有`redirect`选项的 [rewrite](http://wiki.nginx.org/HttpRewriteModule#rewrite) 命令, 例如这段 `nginx.conf` 配置


    rewrite ^ /foo?redirect;  # nginx配置

等价于以下Lua代码


    return ngx.redirect('/foo');  -- Lua代码


而


    rewrite ^ /foo?permanent;  # nginx配置


等价于


    return ngx.redirect('/foo', ngx.HTTP_MOVED_PERMANENTLY)  -- Lua代码


也可以指定URI参数URI, 例如:


    return ngx.redirect('/foo?a=3&b=4')


这个方法将终止当前请求的处理，永不返回. 建议将本方法与 `return` 一起使用, 即, `return ngx.redirect(...)`, 这样意义更明确.

ngx.send_headers
----------------
**语法:** *ok, err = ngx.send_headers()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

显式地发送响应头.

从`v0.8.3` 版本开始这个函数成功时返回 `1` , 出错时 returns `nil` 和 描述错误的字符串.

注意通常并不需要手动发送响应头，因为在[[#ngx.sayngx.say]] 或 [[#ngx.printngx.print]]之前或[[#content_by_luacontent_by_lua]]显式地退出时ngx_lua将自动地发送响应头.

ngx.headers_sent
----------------
**语法:** *value = ngx.headers_sent*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

如果响应头已经被(ngx_lua)发送，返回 `true`，否则返回 `false` .

本命令在 `v0.3.1rc6` 版本中引入.

ngx.print
---------
**语法:** *ok, err = ngx.print(...)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

将参数（作为响应体）发送给客户端. 如果尚未发送响应头，此函数将先发送头，再发送体数据。

从`v0.8.3` 版本开始这个函数成功时返回 `1` , 出错时 returns `nil` 和 描述错误的字符串.

Lua `nil` 值输出为 `"nil"` 字符串 ，Lua boolean 值分别输出为 `"true"` 和 `"false"` 字符串.

支持嵌套的字符串数组，数组中的元素将一个一个地发送:


    local table = {
        "hello, ",
        {"world: ", true, " or ", false,
            {": ", nil}}
    }
    ngx.print(table)


将输出


    hello, world: true or false: nil


非数组table参数将抛出Lua异常.

 <code>ngx.null</code> 常量输出为 <code>"null"</code> 字符串.

这是一个异常调用，立即返回 ，不需要等待所有数据被写进系统发送缓冲区. 要在同步模式中运行，在`ngx.print` 之后调用 `ngx.flush(true)` . 这在做流式输出时特别有用. 详情参阅 [ngx.flush](http://wiki.nginx.org/HttpLuaModule#ngx.flush) .

请注意 `ngx.print` 和 [ngx.say](http://wiki.nginx.org/HttpLuaModule#ngx.say) 都将启动整个 Nginx 输出体过滤器链, 开销比较大. 所以如果在循环中调用它们时要小心，请自行缓冲数据以减少调用次数.

ngx.say
-------
**语法:** *ok, err = ngx.say(...)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

同 [ngx.print](http://wiki.nginx.org/HttpLuaModule#ngx.print) 但多发送一个尾部换行.

ngx.log
-------
**语法:** *ngx.log(log_level, ...)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

用指定的日志级别将参数添加到error.log

Lua `nil` 将输出 `"nil"` 字符串 Lua boolean值输出 `"true"` 或 `"false"` . `ngx.null` 常量也写入为 `"null"` 字符串.

 <code>log_level</code> 参数可以是象<code>ngx.ERR</code> <code>ngx.WARN</code>这样的常量. 详情见 [Nginx日志级别常量](http://wiki.nginx.org/HttpLuaModule#Nginx_log_level_constants).

Nginx 内核中硬编码限制了错误信息长度为 `2048` 字节. 这个限制包括了最后的换行和头部的时间戳。如果消息长度超限，Nginx 会做相应的截断。可以手工修改Nginx代码树中的`src/core/ngx_log.h`文件中的 `NGX_MAX_ERROR_STR` 宏定义来修改这个上限。

ngx.flush
---------
**语法:** *ok, err = ngx.flush(wait?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

将响应数据输出至客户端. 

`ngx.flush` 接受可选的 boolean `wait` 参数(默认值: `false`)，从 `v0.3.1rc34` 版本开始引入。当用默认参数调用时, 会发起一个异步调用 (立即返回， 不需要等待所有数据被写进系统发送缓冲区). 如果 `wait` 参数设为 `true` 则切换到同步模式. 

在同步模式中， 函数只有当所有输出数据全部写入系统发送缓冲区后，或到达 [send_timeout](http://wiki.nginx.org/HttpCoreModule#send_timeout) 设置的超时时限时才返回。注意使用Lua coroutine机制意味着此函数即使在同步模式下也不会阻塞 Nginx 事件循环.

如果在[[#ngx.printngx.print]] 或 [[#ngx.sayngx.say]]之后立即调用 `ngx.flush(true)` , 将导致前者运行于同步模式。这在做流式输出时特别有用.

注意在HTTP 1.0 输出缓冲模式下 `ngx.flush` 无效. 参阅 [HTTP 1.0 支持](http://wiki.nginx.org/HttpLuaModule#HTTP_1.0_support).

从`v0.8.3` 版本开始这个函数成功时返回 `1` , 出错时 returns `nil` 和 描述错误的字符串.

ngx.exit
--------
**语法:** *ngx.exit(status)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

如果 `status >= 200` (即, `ngx.HTTP_OK` 及以上), 它将中断当前请求的执行直接将状态码返回给nginx.

如果`status == 0` (即, `ngx.OK`), 将只退出当前的阶段处理器 ( 如果使用了[[#content_by_luacontent_by_lua]]，则是当前的内容处理器)，会继续执行当前请求的后续阶段（如果有的话）。

 <code>status</code> 参数可以是<code>ngx.OK</code>, <code>ngx.ERROR</code>, <code>ngx.HTTP_NOT_FOUND</code>,
`ngx.HTTP_MOVED_TEMPORARILY`, 或其它[HTTP状态常量](http://wiki.nginx.org/HttpLuaModule#HTTP_status_constants).

要返回带有自定义内容的错误页面, 使用这样的代码:


    ngx.status = ngx.HTTP_GONE
    ngx.say("This is our own content")
    -- 导致退出整个请求而不是当前阶段处理器
    ngx.exit(ngx.HTTP_OK)


看看结果


    $ curl -i http://localhost/test
    HTTP/1.1 410 Gone
    Server: nginx/1.0.6
    Date: Thu, 15 Sep 2011 00:51:48 GMT
    Content-Type: text/plain
    Transfer-Encoding: chunked
    Connection: keep-alive

    This is our own content


可以直接用数字作为参数，例如，


    ngx.exit(501)


注意，虽然这个方法接受所有的 [HTTP状态常量](http://wiki.nginx.org/HttpLuaModule#HTTP_status_constants)作为输入, 但是在[[#core constantscore constants]]中仅支持 `NGX_OK` 和 `NGX_ERROR`。 

我们建议，虽然不强制, 将`return` 语句与此方法一起使用, 如, `return ngx.exit(...)`, 为代码的阅读者以提示.

ngx.eof
-------
**语法:** *ok, err = ngx.eof()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

显式地标出响应输出流的结尾. 在 HTTP 1.1 chunked 编码输出中, 这将导致Nginx核心发送 "last chunk".

如果对下游连接禁止了HTTP 1.1 keep-alive功能, 可以使用此方法让良好实现的HTTP客户端来关闭连接。这个技巧可以用来运行后台任务，而不必让HTTP客户端等待连接，如下例:


    location = /async {
        keepalive_timeout 0;
        content_by_lua '
            ngx.say("got the task!")
            ngx.eof()  -- 良好实现的HTTP客户端将在此时关闭连接
            -- 此处访问 MySQL, PostgreSQL, Redis, Memcached等...
        ';
    }


但是如果你创建了子请求来访问由Nginx上游模块 配置的其它 location，你需要配置此上游模块忽略掉客户端发起的连接断开. 例如，For example, by default the standard [HttpProxyModule](http://wiki.nginx.org/HttpProxyModule)默认会在客户端关闭连接时同时终止子请求和父请求，所以需要在[HttpProxyModule](http://wiki.nginx.org/HttpProxyModule)配置的location中打开 [proxy_ignore_client_abort](http://wiki.nginx.org/HttpProxyModule#proxy_ignore_client_abort) 命令:


    proxy_ignore_client_abort on;


从`v0.8.3` 版本开始这个函数成功时返回 `1` , 出错时 returns `nil` 和 描述错误的字符串.

ngx.sleep
---------
**语法:** *ngx.sleep(seconds)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

非阻塞地睡眠指定的时间. 时间的精度可以到 0.001 seconds (即毫秒).

其内部实现使用Nginx定时器.

从`0.7.20` 版本起, 支持 `0` 作为时间参数.

本命令在 `v0.5.0rc30` 版本中引入.

ngx.escape_uri
--------------
**语法:** *newstr = ngx.escape_uri(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

将 `str` 转义成 URI 片段.

ngx.unescape_uri
----------------
**语法:** *newstr = ngx.unescape_uri(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

将URI片段 `str` 解转义.

例如,


    ngx.say(ngx.unescape_uri("b%20r56+7"))


结果是


    b r56 7


ngx.encode_args
---------------
**语法:** *str = ngx.encode_args(table)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

根据URI编码规则将Lua table编码成一个query string.

例如,


    ngx.encode_args({foo = 3, ["b r"] = "hello world"})


输出


    foo=3&b%20r=hello%20world


table的key必须是 Lua 字符串.

支持多值请求参数. 只需要用Lua table作为参数值, 例如:


    ngx.encode_args({baz = {32, "hello"}})


输出


    baz=32&baz=hello


如果值是空的table，结果等价于 `nil` 值.

支持布尔值作为参数，例如,


    ngx.encode_args({a = true, b = 1})


输出


    a&b=1


如果参数值是`false`, 结果等价于 `nil` 值.

本命令在 `v0.3.1rc27` 版本中引入.

ngx.decode_args
---------------
**语法:** *table = ngx.decode_args(str, max_args?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

将 URI编码的 query-string 解析为Lua table. 此函数是 [ngx.encode_args](http://wiki.nginx.org/HttpLuaModule#ngx.encode_args) 的反函数.

可选的`max_args` 参数指定从 `str` 解析出的参数数量的上限. 默认最多只解析100个请求参数 (包括名称一样的) ，多出来的请求参数将被简单丢弃，以防止DOS攻击.

这个参数设成0表示无上限，会处理接收到的所有的请求参数：


    local args = ngx.decode_args(str, 0)


强烈不建议去掉`max_args` 限制.

本命令在 `v0.5.0rc29` 版本中引入.

ngx.encode_base64
-----------------
**语法:** *newstr = ngx.encode_base64(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

将`str` 进行 base64 编码.

ngx.decode_base64
-----------------
**语法:** *newstr = ngx.decode_base64(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

对`str` 进行base64解码. 如果`str`格式错误将返回`nil`.

ngx.crc32_short
---------------
**语法:** *intval = ngx.crc32_short(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

计算`str`参数的 CRC-32 校验值.

此方法对于相对短的 `str` 参数(小于 30 ~ 60 字节) 比 [ngx.crc32_long](http://wiki.nginx.org/HttpLuaModule#ngx.crc32_long)要快. 结果与 [ngx.crc32_long](http://wiki.nginx.org/HttpLuaModule#ngx.crc32_long) 一样.

其内部实现其实是对Nginx核心`ngx_crc32_short`函数的小小封装。

本命令在 `v0.3.1rc8` 版本中引入.

ngx.crc32_long
--------------
**语法:** *intval = ngx.crc32_long(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

计算`str`参数的 CRC-32 校验值.

此方法对于相对长的 `str` 参数(大于 30 ~ 60 字节) 比 [ngx.crc32_short](http://wiki.nginx.org/HttpLuaModule#ngx.crc32_short)要快.  结果与 [ngx.crc32_short](http://wiki.nginx.org/HttpLuaModule#ngx.crc32_short) 一样.

其内部实现其实是对Nginx核心`ngx_crc32_long`函数的小小封装。

本命令在 `v0.3.1rc8` 版本中引入.

ngx.hmac_sha1
-------------
**语法:** *digest = ngx.hmac_sha1(secret_key, str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

使用密钥`<secret_key>`计算`str`参数的 [HMAC-SHA1](http://en.wikipedia.org/wiki/HMAC)文摘.

此函数返回的是 `HMAC-SHA1` 文摘的二进制数据，如果需要，可以使用[ngx.encode_base64](http://wiki.nginx.org/HttpLuaModule#ngx.encode_base64)将其转成文本形式。

例如,


    local key = "thisisverysecretstuff"
    local src = "some string we want to sign"
    local digest = ngx.hmac_sha1(key, src)
    ngx.say(ngx.encode_base64(digest))


输出


    R/pvxzHC4NLtj7S+kXFg/NePTmk=


本API需要在编译Nginx时打开OpenSSL库 (`./configure`命令加上 `--with-http_ssl_module` 选项).

本命令在 `v0.3.1rc29` 版本中引入.

ngx.md5
-------
**语法:** *digest = ngx.md5(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

返回`str`MD5文摘的16进制文本串.

例如,


    location = /md5 {
        content_by_lua 'ngx.say(ngx.md5("hello"))';
    }


输出


    5d41402abc4b2a76b9719d911017c592


如果需要MD5文摘的二进制数据用 [ngx.md5_bin](http://wiki.nginx.org/HttpLuaModule#ngx.md5_bin).

ngx.md5_bin
-----------
**语法:** *digest = ngx.md5_bin(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

返回`str`MD5文摘的二进制数据.

如果需要十六进制文本串，使用[ngx.md5](http://wiki.nginx.org/HttpLuaModule#ngx.md5).

ngx.sha1_bin
------------
**语法:** *digest = ngx.sha1_bin(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

返回`str`SHA-1文摘的二进制数据.

此函数需要编译Nginx时添加 SHA-1 支持. (通常这意味着在编译Nginx时安装OpenSSL).

本命令在 `v0.5.0rc6` 版本中引入.

ngx.quote_sql_str
-----------------
**语法:** *quoted_value = ngx.quote_sql_str(raw_value)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

根据MySQL的引号规则返回对参数进行转义.

ngx.today
---------
**语法:** *str = ngx.today()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

从nginx缓存的时间中返回当前日期 (格式为`yyyy-mm-dd`) (不象Lua date库，这里没有系统调用).

这是本地时间.

ngx.time
--------
**语法:** *secs = ngx.time()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

从nginx缓存的时间返回从公元纪年到当前时间的秒数 (不象Lua date库，这里没有系统调用).

可以先用[[#ngx.update_timengx.update_time]]强制刷新Nginx时间缓存.

ngx.now
-------
**语法:** *secs = ngx.now()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

从nginx缓存的时间返回浮点数形式的从公元纪年到当前时间的秒数（小数部分是毫秒数） (不象Lua date库，这里没有系统调用).

可以先用[[#ngx.update_timengx.update_time]]强制刷新Nginx时间缓存.

本命令在 `v0.3.1rc32` 版本中引入.

ngx.update_time
---------------
**语法:** *ngx.update_time()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

强制刷新Nginx当前时间缓存. 这个函数包括一个系统调用，有一定的开销，请勿滥用。

本命令在 `v0.3.1rc32` 版本中引入.

ngx.localtime
-------------
**语法:** *str = ngx.localtime()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

从nginx缓存的时间中返回当前时间戳 (格式为`yyyy-mm-dd hh:mm:ss`) (不象Lua 的[os.date](http://www.lua.org/manual/5.1/manual.html#pdf-os.date)函数，这里没有系统调用).

这是本地时间.

ngx.utctime
-----------
**语法:** *str = ngx.utctime()*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

从nginx缓存的时间中返回当前时间戳 (格式为`yyyy-mm-dd hh:mm:ss`) (不象Lua 的[os.date](http://www.lua.org/manual/5.1/manual.html#pdf-os.date)函数，这里没有系统调用).

这是UTC时间.

ngx.cookie_time
---------------
**语法:** *str = ngx.cookie_time(sec)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

返回按cookie超时时间格式化的时间字符串. `sec`参数是以秒为单位的时间戳 (同 [ngx.time](http://wiki.nginx.org/HttpLuaModule#ngx.time) 的返回值).


    ngx.say(ngx.cookie_time(1290079655))
        -- 输出 "Thu, 18-Nov-10 11:27:35 GMT"


ngx.http_time
-------------
**语法:** *str = ngx.http_time(sec)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

返回http头中时间格式的时间字符串 (例如 `Last-Modified` 头). `sec`参数是以秒为单位的时间戳 (同 [ngx.time](http://wiki.nginx.org/HttpLuaModule#ngx.time) 的返回值).


    ngx.say(ngx.http_time(1290079655))
        -- 输出 "Thu, 18 Nov 2010 11:27:35 GMT"


ngx.parse_http_time
-------------------
**语法:** *sec = ngx.parse_http_time(str)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

解析http时间字符串 (象 [ngx.http_time](http://wiki.nginx.org/HttpLuaModule#ngx.http_time) 的返回值) 得到秒数. 如果输入字符串格式不对，返回 `nil` .


    local time = ngx.parse_http_time("Thu, 18 Nov 2010 11:27:35 GMT")
    if time == nil then
        ...
    end


ngx.is_subrequest
-----------------
**语法:** *value = ngx.is_subrequest*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua**

如果当前请求是nginx子请求，返回`true` 否则返回 `false` .

ngx.re.match
------------
**语法:** *captures, err = ngx.re.match(subject, regex, options?, ctx?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

使用Perl兼容的正则表达式`regex`和可选的`options`选项对`subject`字符串进行匹配.

只返回第一个匹配项, 无匹配则返回`nil`. 如果出错，如正则表达式错误或超过了PCRE的栈空间限制, 返回`nil` 和描述错误的字符串.

如果有匹配，返回 Lua table `captures` , 其中 `captures[0]` 装有匹配到的整个子串, `captures[1]` 装有第一个对第一个括号子模板的捕捉结果, `captures[2]` 装有第二个，依此类推.


    local m, err = ngx.re.match("hello, 1234", "[0-9]+")
    if m then
        -- m[0] == "1234"

    else
        if err then
            ngx.log(ngx.ERR, "error: ", err)
            return
        end

        ngx.say("match not found")
    end



    local m, err = ngx.re.match("hello, 1234", "([0-9])[0-9]+")
    -- m[0] == "1234"
    -- m[1] == "1"


从`v0.7.14`版本开始，支持具名捕捉，捕捉的结果用同一个Lua table返回，以key-value的形式与基于下标的捕捉结果在一起。


    local m, err = ngx.re.match("hello, 1234", "([0-9])(?<remaining>[0-9]+)")
    -- m[0] == "1234"
    -- m[1] == "1"
    -- m[2] == "234"
    -- m["remaining"] == "234"


没有匹配上的子模板在`captures` table中的值为`nil`.


    local m, err = ngx.re.match("hello, world", "(world)|(hello)|(?<named>howdy)")
    -- m[0] == "hello"
    -- m[1] == nil
    -- m[2] == "hello"
    -- m[3] == nil
    -- m["named"] == nil


指定`options` 选项来控制如何进行匹配. 支持以下选项:


    a             锚模式(仅从头部开始匹配)

    d             打开 DFA 模式 (最长字元匹配语义).
                  此选项需要 PCRE 6.0+ ，没有将抛出Lua 异常.
                  在 ngx_lua v0.3.1rc30 版本中引入.

    D             打开重复具名模板支持. 此选项允许具名
                  子模板的名称重复, 返回的捕捉结果是一个
                  数组型的 Lua table. 例如,
                    local m = ngx.re.match("hello, world",
                                           "(?<named>\w+), (?<named>\w+)",
                                           "D")
                    -- m["named"] == {"hello", "world"}
                  此选项在 v0.7.14 版本中引入.
                  此选项需要至少 PCRE 8.12版本.

    i             大小写无关模式(类似于Perl的 /i 修饰符)

    j             打开PCRE JIT编译, 需要带有--enable-jit选项编译的 PCRE 8.21+ 
                  . 为了最优的性能,
                  此选项总是应于 'o' 选项一起使用.
                  在 ngx_lua v0.3.1rc30 版本中引入.

    J             打开 PCRE Javascript兼容模式. 此选项在v0.7.14版本中引入 
                  . 此选项需要
                  至少PCRE 8.12版本.

    m             多行模式(类似于Perl /m 修饰符)

    o             编译一次模式 (类似于 Perl /o 修饰符),
                  打开 worker进程级别的正则表达式编译缓存

    s             单行模式 (类似于Perl /s 修饰符)

    u             UTF-8 模式. 此选项要求 PCRE 带
                   --enable-utf8 选项编译，否则将抛出Lua异常.

    U             类似于 "u" 但禁止  PCRE 对目标字符串进行UTF-8校验
                  . 在 ngx_lua v0.8.1 版本中引入.

    x             扩展模式 (类似于Perl /x 修饰符)


以上选项可以组合使用:


    local m, err = ngx.re.match("hello, world", "HEL LO", "ix")
    -- m[0] == "hello"



    local m, err = ngx.re.match("hello, 美好生活", "HELLO, (.{2})", "iu")
    -- m[0] == "hello, 美好"
    -- m[1] == "美好"


 <code>o</code> 选项对性能调优很有用，因为正则模板只编译一次，在worker进程级别进行缓存，被当前Nginx worker进程的所有请求所共享. 可以使用[[#lua_regex_cache_max_entrieslua_regex_cache_max_entries]]来调整正则缓存的上限.

第4个可选参数, `ctx`, 是装有 `pos` 值的Lua table. 如果`ctx` table中指定了 `pos`，则 `ngx.re.match` 将从这个位置开始匹配 . 不管`ctx` table中是否有 `pos` ， `ngx.re.match`在成功匹配后，都会将 `pos` 的值置为整个模板匹配到的子串的后一位. 如果匹配失败, `ctx` table将不变.


    local ctx = {}
    local m, err = ngx.re.match("1234, hello", "[0-9]+", "", ctx)
         -- m[0] = "1234"
         -- ctx.pos == 4



    local ctx = { pos = 2 }
    local m, err = ngx.re.match("1234, hello", "[0-9]+", "", ctx)
         -- m[0] = "34"
         -- ctx.pos == 4


使用`ctx` 参数和 `a` 正则修饰符的组合可以在 `ngx.re.match` 基础上实现词法分析器.

注意，如果指定了`ctx`参数， `options` 参数就是必选的，即使不需要有意义的正则选项，也要提供空的Lua字符串 (`""`).

此方法需要Nginx编译时加上 PCRE 库.  ([PCRE特殊序列的已知问题](http://wiki.nginx.org/HttpLuaModule#Special_PCRE_Sequences)).

要确认打开了 PCRE JIT , 给Nginx 或 ngx_openresty's `./configure`加上`--with-debug`选项，打开Nginx debug日志. 然后在`error_log`命令中打开 "debug" 错误日志级别. 如果打开了PCRE JIT，将出现以下信息:


    pcre JIT compiling result: 1


此功能在 `v0.2.1rc11` 版本中引入.

ngx.re.gmatch
-------------
**语法:** *iterator, err = ngx.re.gmatch(subject, regex, options?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

与[ngx.re.match](http://wiki.nginx.org/HttpLuaModule#ngx.re.match)类似, 但返回 a Lua iterator, 使开发者可以遍历所有PCRE `regex`对`<subject>`字符串的匹配结果.

如果出错，如正则表达式格式不正确, 返回`nil` 和描述错误信息的字符串.

以下是对基本用法的示例:


    local iterator, err = ngx.re.gmatch("hello, world!", "([a-z]+)", "i")
    if not iterator then
        ngx.log(ngx.ERR, "error: ", err)
        return
    end

    local m
    m, err = iterator()    -- m[0] == m[1] == "hello"
    if err then
        ngx.log(ngx.ERR, "error: ", err)
        return
    end

    m, err = iterator()    -- m[0] == m[1] == "world"
    if err then
        ngx.log(ngx.ERR, "error: ", err)
        return
    end

    m, err = iterator()    -- m == nil
    if err then
        ngx.log(ngx.ERR, "error: ", err)
        return
    end


更多的时候我们只要将其放入一个Lua循环:


    local it, err = ngx.re.gmatch("hello, world!", "([a-z]+)", "i")
    if not it then
        ngx.log(ngx.ERR, "error: ", err)
        return
    end

    while true do
        local m, err = it()
        if err then
            ngx.log(ngx.ERR, "error: ", err)
            return
        end

        if not m then
            -- no match found (any more)
            break
        end

        -- found a match
        ngx.say(m[0])
        ngx.say(m[1])
    end


可选的`options`参数与 [ngx.re.match](http://wiki.nginx.org/HttpLuaModule#ngx.re.match) 方法中的参数语义完全一致。

目前的实现要求返回的iterator只能用于一个请求. 也就是说， *不要* 把它赋值给一个属于持久命名空间如Lua package的变量.

此方法需要Nginx编译时加上 PCRE 库.  ([PCRE特殊序列的已知问题](http://wiki.nginx.org/HttpLuaModule#Special_PCRE_Sequences)).

本功能在 `v0.2.1rc12` 版本中引入.

ngx.re.sub
----------
**语法:** *newstr, n, err = ngx.re.sub(subject, regex, replace, options?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

将 Perl 兼容的正则表达式 `regex` 对 `subject` 字符串参数的第一个匹配替换为字符串或函数参数 `replace`. 可选参数 `options` 与 [ngx.re.match](http://wiki.nginx.org/HttpLuaModule#ngx.re.match) 中的语义一致.

返回生成的新字符串以及成功匹配的数目. 如果出错, 如正则表达式语法错或 `<replace>` 格式错, 返回 `nil` 和描述错误的字符串.

如果 `replace` 是航空器, 则它将被用作字符串替换的特殊模板. 例如,


    local newstr, n, err = ngx.re.sub("hello, 1234", "([0-9])[0-9]", "[$0][$1]")
    if newstr then
        -- newstr == "hello, [12][1]34"
        -- n == 1
    else
        ngx.log(ngx.ERR, "error: ", err)
        return
    end


其中 `$0` 指代匹配的整个子串， `$1` 指代收第一个括号捕捉的子串.

可以用大括号来消除变量名与后台字符量的歧义: 


    local newstr, n, err = ngx.re.sub("hello, 1234", "[0-9]", "${0}00")
        -- newstr == "hello, 10034"
        -- n == 1


`replace`中的美元符 (`$`)可以再用一个美元符进行转义，例如,


    local newstr, n, err = ngx.re.sub("hello, 1234", "[0-9]", "$$")
        -- newstr == "hello, $234"
        -- n == 1


还要用反斜杠对美元符进行转义.

如果 `replace` 参数的类型是 "函数", 则它会作用于 "匹配结果table" 生成替换后的字符串.  `replace` 函数的 `replace` 参数与 [ngx.re.match](http://wiki.nginx.org/HttpLuaModule#ngx.re.match) 的返回值完全一样. 以下是示例:


    local func = function (m)
        return "[" .. m[0] .. "][" .. m[1] .. "]"
    end
    local newstr, n, err = ngx.re.sub("hello, 1234", "( [0-9] ) [0-9]", func, "x")
        -- newstr == "hello, [12][1]34"
        -- n == 1


用作 `replace` 函数参数的[[#ngx.re.matchngx.re.match]]返回值中的美元符并不是特殊字符.

此方法需要Nginx编译时加上 PCRE 库.  ([PCRE特殊序列的已知问题](http://wiki.nginx.org/HttpLuaModule#Special_PCRE_Sequences)).

本功能在 `v0.2.1rc13` 版本中引入.

ngx.re.gsub
-----------
**语法:** *newstr, n, err = ngx.re.gsub(subject, regex, replace, options?)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

类似 [ngx.re.sub](http://wiki.nginx.org/HttpLuaModule#ngx.re.sub), 但是做全局替换.

示例:


    local newstr, n, err = ngx.re.gsub("hello, world", "([a-z])[a-z]+", "[$0,$1]", "i")
    if newstr then
        -- newstr == "[hello,h], [world,w]"
        -- n == 2
    else
        ngx.log(ngx.ERR, "error: ", err)
        return
    end



    local func = function (m)
        return "[" .. m[0] .. "," .. m[1] .. "]"
    end
    local newstr, n, err = ngx.re.gsub("hello, world", "([a-z])[a-z]+", func, "i")
        -- newstr == "[hello,h], [world,w]"
        -- n == 2


此方法需要Nginx编译时加上 PCRE 库.  ([PCRE特殊序列的已知问题](http://wiki.nginx.org/HttpLuaModule#Special_PCRE_Sequences)).

本功能在 `v0.2.1rc15` 版本中引入.

ngx.shared.DICT
---------------
**语法:** *dict = ngx.shared.DICT*

**语法:** *dict = ngx.shared[name_var]*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

获取由 [[#lua_shared_dictlua_shared_dict]]命令定义的基于共享内存的Lua字典对象 `DICT` .

返回的 `dict` 对象有以下方法:

* [get](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.get)
* [get_stale](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.get_stale)
* [set](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.set)
* [safe_set](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.safe_set)
* [add](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.add)
* [safe_add](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.safe_add)
* [replace](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.replace)
* [incr](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.incr)
* [delete](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.delete)
* [flush_all](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.flush_all)
* [flush_expired](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.flush_expired)

以下是示例:


    http {
        lua_shared_dict dogs 10m;
        server {
            location /set {
                content_by_lua '
                    local dogs = ngx.shared.dogs
                    dogs:set("Jim", 8)
                    ngx.say("STORED")
                ';
            }
            location /get {
                content_by_lua '
                    local dogs = ngx.shared.dogs
                    ngx.say(dogs:get("Jim"))
                ';
            }
        }
    }


我们来测试一下:


    $ curl localhost/set
    STORED

    $ curl localhost/get
    8

    $ curl localhost/get
    8


不管有多少个Nginx worker进程，访问`/get`时每次都输出 `8`  是因为 `dogs` 字典位于共享内存中，对 *所有* worker进程可见。

这部分共享字典的内容即使在服务器重新加载配置时(通过向Nginx进程发送 `HUP` 信号或者使用 `-s reload` 命令行参数)仍能保持.

但是如果Nginx服务器退出了，此字典数据就没有了。

本功能在 `v0.3.1rc22` 版本中引入.

ngx.shared.DICT.get
-------------------
**语法:** *value, flags = ngx.shared.DICT:get(key)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

获取字典 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT) 中键 `key` 所对应的值. 如果键不存在或已过期，返回 `nil` .

返回的值拥有其被写入字典时同样的数据类型，例如, Lua 布尔值, 数字, 或字符串.

这个方法的第一个参数必须是字典对象本身, 例如,


    local cats = ngx.shared.cats
    local value, flags = cats.get(cats, "Marry")


或使用Lua方法调用的语法糖:


    local cats = ngx.shared.cats
    local value, flags = cats:get("Marry")


这两种形式本质上是一样的.

如果用户标记是 `0` (默认值), 将不会返回标记值.

本功能在 `v0.3.1rc22` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.get_stale
-------------------------
**语法:** *value, flags, stale = ngx.shared.DICT:get_stale(key)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

类似于 [get](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.get) 但即使键过期了，仍返回其值.

同时返回第3个值, `stale`, 指明此键是否已过期.

注意已过期的键的值并不能保证可用，所以不能依赖于过期项的可用性。.

本命令在 `v0.8.6` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.set
-------------------
**语法:** *success, err, forcible = ngx.shared.DICT:set(key, value, exptime?, flags?)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

无条件向基于共享内存的字典 [[#ngx.shared.DICTngx.shared.DICT]]中写入一个key-value对. 返回3个值:

* `success`: 布尔值，标明是否成功写入.
* `err`: 文本错误信息, 可以是 `"no memory"`.
* `forcible`: 布尔值，标明是否由于共享内存区的空间不够而导致其它有效项被删除.

被插入的 `value` 参数可以是 Lua布尔值, 数字，字符串, 或r `nil`. 类型信息也会被存入字典，这样后续调用[[#ngx.shared.DICT.getget]]时可以取出同样类型的数据.

可选参数 `exptime` 指定插入的key-value对的过期时间 (单位是秒). 时间精度为 `0.001` 秒. 如果 `exptime` 值为 `0` (默认值), 此项将永不过期.

可选的 `flags` 参数是与被插入的项相关的用户标志。也可以在以后与值一起获取到. 用户标记在实现上是以无符号 32位整数存储的. 默认值是 `0`. 用户标记参数在 `v0.5.0rc2` 版本中引入。

如果为当前的键－值对项分配内存失败, `set` 将尝试用LRU算法从存储区删除某些已经存在的项. 注意，在这里LRU比过期时间优先级高。如果已经删除了几十项，剩下的存储空间仍然不够 (受到 [lua_shared_dict](http://wiki.nginx.org/HttpLuaModule#lua_shared_dict) 或内存段的限制),  `err` 的返回值将为`no memory` ， `success` 为 `false`.

如果此方法通过采取LRU的方式从字典中删除了一些尚未过期的项, `forcible` 返回值为 `true`. 如果成功加入了存储项而没有删除其它的有效项,  `forcible` 返回值为 `false`.

这个方法的第一个参数必须是字典对象本身, 例如,


    local cats = ngx.shared.cats
    local succ, err, forcible = cats.set(cats, "Marry", "it is a nice cat!")


或使用Lua方法调用的语法糖:


    local cats = ngx.shared.cats
    local succ, err, forcible = cats:set("Marry", "it is a nice cat!")


这两种形式本质上是一样的.

本功能在 `v0.3.1rc22` 版本中引入.

请注意虽然在内部实现中key-value对的写入是原子操作，但其原子性并不跨越方法调用的边界.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.safe_set
------------------------
**语法:** *ok, err = ngx.shared.DICT:safe_set(key, value, exptime?, flags?)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

与 [set](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.set) 类似, 但在共享存储区空间不够时不会(以LRU的方式)覆盖未过期的项. 在这种情况下，它将立即返回 `nil` 和错误信息 "no memory".

本功能在 `v0.7.18` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.add
-------------------
**语法:** *success, err, forcible = ngx.shared.DICT:add(key, value, exptime?, flags?)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

类似于 [set](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.set) , 但只有当键值 *不* 存在时才将key-value对存入 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT) .

如果字典中已经存在 `key` 键（而且尚未过期）, `success` 将返回 `false` ， `err` 将返回 `"exists"`.

本功能在 `v0.3.1rc22` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.safe_add
------------------------
**语法:** *ok, err = ngx.shared.DICT:safe_add(key, value, exptime?, flags?)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

类似于 [add](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.add), 但当共享内存区空间不足时不会（以LRU的方式）覆盖未过期的项. 在这种情况下，它将立即返回 `nil` 和错误信息 "no memory".

本功能在 `v0.7.18` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.replace
-----------------------
**语法:** *success, err, forcible = ngx.shared.DICT:replace(key, value, exptime?, flags?)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

类似于 [set](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.set) , 但只有在字典[[#ngx.shared.DICTngx.shared.DICT]]中 *不* 存在键值的时候才将key-value对写入.

如果字典中 *没有* `key` 键 (或已过期), `success` 将返回 `false` `err` 将返回 `"not found"`.

本功能在 `v0.3.1rc22` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.delete
----------------------
**语法:** *ngx.shared.DICT:delete(key)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

从基于共享内存的字典[[#ngx.shared.DICTngx.shared.DICT]]中无条件删除key-value对.

等价于 `ngx.shared.DICT:set(key, nil)`.

本功能在 `v0.3.1rc22` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.incr
--------------------
**语法:** *newval, err = ngx.shared.DICT:incr(key, value)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

为基于共享内存的字典[[#ngx.shared.DICTngx.shared.DICT]]中`key`键的数值加上 `value`. 如果成功，返回结果值否则返回 `nil` 和错误信息.

键必须在字典中已经存在，否则返回 `nil` 和 `"not found"`.

如果原来的值不是有效的Lua 数值, 返回 `nil` 和 `"not a number"`.

 <code>value</code> 参数可以是任何有效的 Lua 数值, 如负数或浮点数.

本功能在 `v0.3.1rc22` 版本中引入.

参阅 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.flush_all
-------------------------
**语法:** *ngx.shared.DICT:flush_all()*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

清除字典中所有项目. 此方法事实上不会释放字典中所有的内存块，只是让所有现有的项置过期。

本功能在 `v0.5.0rc17` 版本中引入.

参阅 [ngx.shared.DICT.flush_expired](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.flush_expired) 和 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.flush_expired
-----------------------------
**语法:** *flushed = ngx.shared.DICT:flush_expired(max_count?)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

消除字典中过期的项, 最多清除 `max_count` 项. 如果 `max_count` 参数没有或是 `0`, 那就是清除所有的. 返回实际被清除的项的数目。

与 [flush_all](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.flush_all) 不同, 本方法会事实上释放已过期项目占用的内存.

本功能在 `v0.6.3` 版本中引入.

参阅 [ngx.shared.DICT.flush_all](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT.flush_all) 和 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT).

ngx.shared.DICT.get_keys
------------------------
**语法:** *keys = ngx.shared.DICT:get_keys(max_count?)*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

从字典中返回键的列表, 数量最大为 `<max_count>`.

默认只返回前1024个值 (如果有的话. 如果 `<max_count>` 参数是值是 `0`, 即使有多于1024项，也全部返回.

**警告** 对键真的很多的字典要小心使用本方法。这个方法可能会将字典锁一段时间，阻塞试图访问此字典的所有nginx worker进程。

本功能在 `v0.7.3` 版本中引入.

ngx.socket.udp
--------------
**语法:** *udpsock = ngx.socket.udp()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

创建并返回一个UDP或基于数据报的 unix domain socket 对象(也是 "cosocket" 的一种). 该对象支持以下方法:

* [setpeername](http://wiki.nginx.org/HttpLuaModule#udpsock:setpeername)
* [send](http://wiki.nginx.org/HttpLuaModule#udpsock:send)
* [receive](http://wiki.nginx.org/HttpLuaModule#udpsock:receive)
* [close](http://wiki.nginx.org/HttpLuaModule#udpsock:close)
* [settimeout](http://wiki.nginx.org/HttpLuaModule#udpsock:settimeout)

它被设计为与 [LuaSocket](http://w3.impa.br/~diego/software/luasocket/udp.html)库的 UDP API兼容，但是它天生是100%非阻塞的。

本功能在 `v0.5.7` 版本中引入.

参阅 [ngx.socket.tcp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp).

udpsock:setpeername
-------------------
**语法:** *ok, err = udpsock:setpeername(host, port)*

**语法:** *ok, err = udpsock:setpeername("unix:/path/to/unix-domain.socket")*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

尝试将一个 UDP socket 对象连接到一个远程服务器或者数据报的unix domain socket文件. 由于数据报协议事实上是无连接的，本方法并不实际创建 "连接", 只是设置远程对象的名字，以备后续的 读/写 操作使用.

 <code>host</code> 参数可以是IP地址或域名. 如果是域名, 本方法将使用Nginx 核心的 动态域名解析器非阻塞地解析域名，要求在 <code>nginx.conf</code> 文件里使用[resolver](http://wiki.nginx.org/HttpCoreModule#resolver) 命令，如:


    resolver 8.8.8.8;  # 使用 Google 的公开 DNS 服务器


如果从域名解析出来多个IP地址，本方法将从中随机选取一个.

如果出错, 返回 `nil` 和描述错误的字符串. 如果成功, 返回 `1`.

以下是连接到 UDP (memcached) 服务器的例子:


    location /test {
        resolver 8.8.8.8;

        content_by_lua '
            local sock = ngx.socket.udp()
            local ok, err = sock:setpeername("my.memcached.server.domain", 11211)
            if not ok then
                ngx.say("failed to connect to memcached: ", err)
                return
            end
            ngx.say("successfully connected to memcached!")
            sock:close()
        ';
    }


从 `v0.7.18` 版本开始, 也可以连接到Linux上的数据报unix domain socket 文件:


    local sock = ngx.socket.udp()
    local ok, err = sock:setpeername("unix:/tmp/some-datagram-service.sock")
    if not ok then
        ngx.say("failed to connect to the datagram unix domain socket: ", err)
        return
    end


假设数据报服务监听的是 unix domain socket 文件`/tmp/some-datagram-service.sock` 客户端socket会使用Linux的 "自动绑定" 功能.

对已经连接的socket对象调用此方法将导致之前的连接先被关闭.

本命令在 `v0.5.7` 版本中引入.

udpsock:send
------------
**语法:** *ok, err = udpsock:send(data)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

在当前的 UDP 或数据报unix domain socket对象上发送数据.

成功则返回`1`. 否则返回 `nil` 和描述错误的字符串.

输入参数 `data` 可以是 Lua 字符串或存有字符串片段的 (嵌套) Lua table. 如果是table参数， 本方法会将所有的字符串元素一块一块地拷贝到下层Nginx socket发送缓冲区，通常这比在Lua里进行字符串拼接要快。

本功能在 `v0.5.7` 版本中引入.

udpsock:receive
---------------
**语法:** *data, err = udpsock:receive(size?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

从UDP或数据报unix domain socket对象接收数据，可用 `size` 指定缓冲区大小.

这是一个同步的，100%非阻塞的操作.

如果成功则返回接收到的数据; 如果出错则返回 `nil` 和描述错误的字符串.

如果指定了 `size` 参数, 本方法将用此数值作为接收缓冲区的大小. 但如果这个值超过了 `8192`, 缓冲区大小为 `8192` .

如果没有参数，缓冲区大小默认为 `8192` .

读操作的超时由 [lua_socket_read_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_read_timeout) 配置命令和 [settimeout](http://wiki.nginx.org/HttpLuaModule#udpsock:settimeout) 方法控制. 后者具有更高的优先级. 例如:


    sock:settimeout(1000)  -- 1秒超时
    local data, err = sock:receive()
    if not data then
        ngx.say("failed to read a packet: ", data)
        return
    end
    ngx.say("successfully read a packet: ", data)


要调用 [settimeout](http://wiki.nginx.org/HttpLuaModule#udpsock:settimeout) 必须在调用本方法 *之前* .

本功能在 `v0.5.7` 版本中引入.

udpsock:close
-------------
**语法:** *ok, err = udpsock:close()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

关闭当前的UDP或数据报unix domain socket. 成功返回 `1` 失败返回 `nil` 和描述错误的字符串.

没有调用此方法的Socket对象 (及相关的连接) 会被Lua GC或当前客户端HTTP请求结束处理过程释放.

本功能在 `v0.5.7` 版本中引入.

udpsock:settimeout
------------------
**语法:** *udpsock:settimeout(time)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

设置后续socket操作(如 [[#udpsock:receivereceive]])的超时时间，以毫秒计.

用本方法作的设置优先级高于使用配置命令如 [lua_socket_read_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_read_timeout)等.

本功能在 `v0.5.7` 版本中引入.

ngx.socket.tcp
--------------
**语法:** *tcpsock = ngx.socket.tcp()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

创建并返回一个TCP或流unix domain socket对象 (也是 "cosocket" 的一种). 该对象支持以下方法:

* [connect](http://wiki.nginx.org/HttpLuaModule#tcpsock:connect)
* [send](http://wiki.nginx.org/HttpLuaModule#tcpsock:send)
* [receive](http://wiki.nginx.org/HttpLuaModule#tcpsock:receive)
* [close](http://wiki.nginx.org/HttpLuaModule#tcpsock:close)
* [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout)
* [setoption](http://wiki.nginx.org/HttpLuaModule#tcpsock:setoption)
* [receiveuntil](http://wiki.nginx.org/HttpLuaModule#tcpsock:receiveuntil)
* [setkeepalive](http://wiki.nginx.org/HttpLuaModule#tcpsock:setkeepalive)
* [getreusedtimes](http://wiki.nginx.org/HttpLuaModule#tcpsock:getreusedtimes)

它被设计为与 [LuaSocket](http://w3.impa.br/~diego/software/luasocket/tcp.html)库的 TCP API兼容，但是它天生是100%非阻塞的。我们还引入了一些新的API来提供更多的功能.

本功能在 `v0.5.1rc1` 版本中引入.

参阅 [ngx.socket.udp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.udp).

tcpsock:connect
---------------
**语法:** *ok, err = tcpsock:connect(host, port, options_table?)*

**语法:** *ok, err = tcpsock:connect("unix:/path/to/unix-domain.socket", options_table?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

尝试将一个TCP socket对象与远程服务器或流unix domain socket文件进行不阻塞连接。

在实际解析域名和连接到远程实体之前, 本方法总会查找连接池中之前用本方法(或[ngx.socket.connect](http://wiki.nginx.org/HttpLuaModule#ngx.socket.connect) 函数)创建的空闲连接.

 <code>host</code> 参数可以是IP地址或域名. 如果是域名, 本方法将使用Nginx 核心的 动态域名解析器非阻塞地解析域名，要求在 <code>nginx.conf</code> 文件里使用[resolver](http://wiki.nginx.org/HttpCoreModule#resolver) 命令，如:


    resolver 8.8.8.8;  # 使用 Google 的公开 DNS 服务器


如果从域名解析出来多个IP地址，本方法将从中随机选取一个.

如果出错, 返回 `nil` 和描述错误的字符串. 如果成功, 返回 `1`.

以下 连接到 TCP 服务器的例子:


    location /test {
        resolver 8.8.8.8;

        content_by_lua '
            local sock = ngx.socket.tcp()
            local ok, err = sock:connect("www.google.com", 80)
            if not ok then
                ngx.say("failed to connect to google: ", err)
                return
            end
            ngx.say("successfully connected to google!")
            sock:close()
        ';
    }


也可以连接到Unix domain socket文件:


    local sock = ngx.socket.tcp()
    local ok, err = sock:connect("unix:/tmp/memcached.sock")
    if not ok then
        ngx.say("failed to connect to the memcached unix domain socket: ", err)
        return
    end


假设 memcached (或其它的什么东西) 正监听 unix domain socket 文件`/tmp/memcached.sock`.

连接操作的超时由 [lua_socket_connect_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_connect_timeout) 配置命令和 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 控制. 后者具有更高的优先级. 例如:


    local sock = ngx.socket.tcp()
    sock:settimeout(1000)  -- 1秒超时
    local ok, err = sock:connect(host, port)


要调用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 必须在调用本方法 *之前* .

对已经连接的socket对象调用此方法将导致之前的连接先被关闭.

可以指定一个可选的Lua table作为最后一个参数来提供各种连接选项:

* `pool`
	自定义要使用的连接池的名称. 如果没有提供, 连接池的名称会以 `"<host>:<port>"` 或 `"<unix-socket-path>"`的格式自动生成.

这个可选的table参数在 `v0.5.7` 版本中引入.

本命令在 `v0.5.0rc1` 版本中引入.

tcpsock:send
------------
**语法:** *bytes, err = tcpsock:send(data)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

发送数据，不阻塞当前TCP或Unix domain socket连接.

本方法是个同步操作，直到 *所有*数据被写入系统socket发送缓冲区，或者出错时才返回.

如果成功，则返回发送的总的字节数。否则返回 `nil` 和描述错误的字符串.

输入参数 `data` 可以是 Lua 字符串或存有字符串片段的 (嵌套) Lua table. 如果是table参数， 本方法会将所有的字符串元素一块一块地拷贝到下层Nginx socket发送缓冲区，通常这比在Lua里进行字符串拼接要快。

发送操作的超时由 [lua_socket_send_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_send_timeout) 配置命令和 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 管理. 后者具有更高的优先级. 例如:


    sock:settimeout(1000)  -- 1秒超时
    local bytes, err = sock:send(request)


要调用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 必须在调用本方法 *之前* .

本功能在 `v0.5.1rc1` 版本中引入.

tcpsock:receive
---------------
**语法:** *data, err, partial = tcpsock:receive(size)*

**语法:** *data, err, partial = tcpsock:receive(pattern?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

根据指定的读取模式或大小从已连接的socket中读取数据.

与[[#tcpsock:sendsend]]一样，本方法是一个同步的100％非阻塞的操作.

如果成功则返回接收到的数据; 如果出错则返回 `nil` 和描述错误的字符串以及接收到的部分数据.

如果输入参数是数值 (或是长得象数值的字符串), 它将被理解为大小. 本方法将在读取到这个数值大小的数据或出错时才返回.

如果指定的是一个长得不象数值的字符串, 它将被理解成一种 "模式". 支持以下模式:

* `'*a'`: 从socket中读取，直到连接被关闭. 不会执行行结束符转换;
* `'*l'`: 从socket中按行读取文本. 每一行以 `换行` (LF) 符(ASCII 10)（之前可能有一个 `回车` (CR) 符(ASCII 13)）结束. 在返回的行文本中不包括回车和换行. 事实上，模式会忽略所有的回车符.

如果没有给参数，本方法默认以 `'*l'` 即按行读取模式执行.

读操作的超时由 [lua_socket_read_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_read_timeout) 配置命令和 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 方法控制. 后者具有更高的优先级. 例如:


    sock:settimeout(1000)  -- 1秒超时
    local line, err, partial = sock:receive()
    if not line then
        ngx.say("failed to read a line: ", err)
        return
    end
    ngx.say("successfully read a line: ", line)


要调用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 必须在调用本方法 *之前* .

本功能在 `v0.5.1rc1` 版本中引入.

tcpsock:receiveuntil
--------------------
**语法:** *iterator = tcpsock:receiveuntil(pattern, options?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

本方法返回一个iterator Lua函数，此函数被用来读取数据流，直到遇到指定的模式或出错。

以下是使用本方法来用边界序列 `--abcedhb` 读取数据:


    local reader = sock:receiveuntil("\r\n--abcedhb")
    local data, err, partial = reader()
    if not data then
        ngx.say("failed to read the data stream: ", err)
    end
    ngx.say("read the data stream: ", data)


如果不加任何参数调用iterator函数，它将返回输入数据流中在指定的模式字符串 *之前* 的数据. 所以在上面的例子中, 如果输入数据流是 `'hello, world!-agentzh\r\n--abcedhb blah blah'`, 则将返回 `'hello, world!-agentzh'` .

如果出错，iterator函数返回 `nil` 和一个描述错误的字符串，以及目前读取的部分数据.

iterator函数可以多次调用，并且可以安全地与其它cosocket方法或其它iterator函数混合调用.

如果带有`size`参数， iterator函数的行为略有不同  (象一个真正的iterator) . 对它的每次调用都会读取 `size` 字节的数据，最后一次调用（遇到边界模式或出错）返回 `nil` . 对iterator函数的最后一次成功的调用, `err` 返回值为 `nil` too. 在最后一次成功返回`nil`数据和`nil`错误后，iterator函数将被重置。考虑下面的例子:


    local reader = sock:receiveuntil("\r\n--abcedhb")

    while true do
        local data, err, partial = reader(4)
        if not data then
            if err then
                ngx.say("failed to read the data stream: ", err)
                break
            end

            ngx.say("read done")
            break
        end
        ngx.say("read chunk: [", data, "]")
    end


对于输入数据流 `'hello, world!-agentzh\r\n--abcedhb blah blah'`, 我们将得到这样的输出:


    read chunk: [hell]
    read chunk: [o, w]
    read chunk: [orld]
    read chunk: [! -a]
    read chunk: [gent]
    read chunk: [zh]
    read done


注意，如果用来做流解析的边界模式有歧义，实际返回的数据 *可能* 会比`size`参数指定的长度略长. 而在接近数据流边界处, 实际返回的数据也可能比指定的大小小.

iterator 函数读操作的超时由 [lua_socket_read_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_read_timeout) 配置命令和 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 方法控制. 后者具有更高的优先级. 例如:


    local readline = sock:receiveuntil("\r\n")

    sock:settimeout(1000)  -- 1秒超时
    line, err, partial = readline()
    if not line then
        ngx.say("failed to read a line: ", err)
        return
    end
    ngx.say("successfully read a line: ", line)


重要的是如果要调用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) ， 一定要在调用iterator函数*之前*  (注意这里 `receiveuntil` 的调用次序无所谓).

从`v0.5.1` 版本起, 本方法还接受可选的 `options` table参数来控制行为. 支持以下选项:

* `inclusive`

 <code>inclusive</code> 是一个布尔值，用来控制在返回的数据字符串中是否包括模式串. 默认为<code>false</code>. 例如,


    local reader = tcpsock:receiveuntil("_END_", { inclusive = true })
    local data = reader()
    ngx.say(data)


对于输入数据流 `"hello world _END_ blah blah blah"`, 上面的例子将输出 `hello world _END_`, 包括模式串`_END_` 本身.

本命令在 `v0.5.0rc1` 版本中引入.

tcpsock:close
-------------
**语法:** *ok, err = tcpsock:close()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

关闭当前 TCP 或流 unix domain socket. 成功返回 `1` 失败返回 `nil` 和描述错误的字符串.

注意不需要对调用过[[#tcpsock:setkeepalivesetkeepalive]]方法的socket对象调用此方法，因为socket对象已经关闭了 (相应的连接被保存进内置的连接池).

没有调用此方法的Socket对象 (及相关的连接) 会被Lua GC或当前客户端HTTP请求结束处理过程释放.

本功能在 `v0.5.1rc1` 版本中引入.

tcpsock:settimeout
------------------
**语法:** *tcpsock:settimeout(time)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

以毫秒为单位设置后续 socket操作([[#tcpsock:connectconnect]], [[#tcpsock:receivereceive]], and iterators returned from [[#tcpsock:receiveuntilreceiveuntil]])的超时时间.

使用本方法作的设置优先级高于使用配置命令, 包括 [lua_socket_connect_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_connect_timeout), [lua_socket_send_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_send_timeout), 和 [lua_socket_read_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_read_timeout).

注意本方法 *不会* 影响 [lua_socket_keepalive_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_keepalive_timeout) 设置; 必须通过 [[#tcpsock:setkeepalivesetkeepalive]]的`timeout` 参数来达到这个目的。

本功能在 `v0.5.1rc1` 版本中引入.

tcpsock:setoption
-----------------
**语法:** *tcpsock:setoption(option, value?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

加入此函数是为了兼容[LuaSocket](http://w3.impa.br/~diego/software/luasocket/tcp.html) API，它什么也不做. 其功能将来会实现.

本功能在 `v0.5.1rc1` 版本中引入.

tcpsock:setkeepalive
--------------------
**语法:** *ok, err = tcpsock:setkeepalive(timeout?, size?)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

立即将当前socket的连接存入 cosocket 内置连接池，保持其存活状态，直接对其再次调用 [connect](http://wiki.nginx.org/HttpLuaModule#tcpsock:connect) 或设置的最大空闲时间到期了.

第一个可选参数, `timeout`, 可以用来指定当前连接的最大空闲时间 (毫秒) . 如果没有, 将使用[lua_socket_keepalive_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_keepalive_timeout) 配置命令中的默认设置. 如果其值是 `0` , 则没有超时时限.

第二个可选参数, `size`, 可以用来指定当前服务器(当前的主机-端口对，或 unix domain socket文件路径)连接池中的最大连接数. 注意当连接池一旦被创建，将不能改变其大小。如果没有这个参数，将使用 [lua_socket_pool_size](http://wiki.nginx.org/HttpLuaModule#lua_socket_pool_size) 配置命令中的默认设置.

当连接池使用超限时, 池中最不经常使用的 (空闲) 连接将被关闭，为当前连接腾出空间。

注意cosocket连接池是每个nginx worker进程所有的而不是整个nginx服务器进程所有，所以这里所设置的上限也适用于每个单独的nginx worker 进程.

池中的空闲连接上的任何异常事件都将被监视，如连接中断或意外的输入数据，这时出问题的连接将被关闭并从池中删除。

成功返回 `1`; 否则返回 `nil` 和描述错误的字符串.

此方法将使当前cosocket对象进入 "closed" 状态, 所以后续不需要对其调用 [close](http://wiki.nginx.org/HttpLuaModule#tcpsock:close) .

本功能在 `v0.5.1rc1` 版本中引入.

tcpsock:getreusedtimes
----------------------
**语法:** *count, err = tcpsock:getreusedtimes()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

本方法返回当前连接被 (成功) 使用的次数. 出错时, 返回`nil` 和描述错误的字符串.

如果当前连接不是从内置连接池来的, 本方法总是返回 `0`, 表示此连接（尚）未被重用过. 如果连接是从内置连接池中取出的，则返回值总是非0的. 所以此方法也可以用来确定当前连接是否是从池中来的.

本功能在 `v0.5.1rc1` 版本中引入.

ngx.socket.connect
------------------
**语法:** *tcpsock, err = ngx.socket.connect(host, port)*

**语法:** *tcpsock, err = ngx.socket.connect("unix:/path/to/unix-domain.socket")*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

此函数是将 [ngx.socket.tcp()](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp) 和 [connect()](http://wiki.nginx.org/HttpLuaModule#tcpsock:connect) 组合到一个操作中的简写. 它的实际实现象这样:


    local sock = ngx.socket.tcp()
    local ok, err = sock:connect(...)
    if not ok then
        return nil, err
    end
    return sock


不能使用 [settimeout](http://wiki.nginx.org/HttpLuaModule#tcpsock:settimeout) 来设置本方法的连接超时，必须用 [lua_socket_connect_timeout](http://wiki.nginx.org/HttpLuaModule#lua_socket_connect_timeout) 命令在配置阶段设置.

本功能在 `v0.5.1rc1` 版本中引入.

ngx.get_phase
-------------
**语法:** *str = ngx.get_phase()*

**可包含于:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

返回当前运行的阶段名称. 可能的返回值有

* `init`
	可能是 [init_by_lua](http://wiki.nginx.org/HttpLuaModule#init_by_lua) 或 [init_by_lua_file](http://wiki.nginx.org/HttpLuaModule#init_by_lua_file) 上下文.
* `set`
	可能是 [set_by_lua](http://wiki.nginx.org/HttpLuaModule#set_by_lua) 或 [set_by_lua_file](http://wiki.nginx.org/HttpLuaModule#set_by_lua_file) 上下文.
* `rewrite`
	可能是 [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) 或 [rewrite_by_lua_file](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua_file) 上下文.
* `access`
	可能是 [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua) or [access_by_lua_file](http://wiki.nginx.org/HttpLuaModule#access_by_lua_file) 上下文.
* `content`
	可能是 [content_by_lua](http://wiki.nginx.org/HttpLuaModule#content_by_lua) or [content_by_lua_file](http://wiki.nginx.org/HttpLuaModule#content_by_lua_file) 上下文.
* `header_filter`
	可能是 [header_filter_by_lua](http://wiki.nginx.org/HttpLuaModule#header_filter_by_lua) or [header_filter_by_lua_file](http://wiki.nginx.org/HttpLuaModule#header_filter_by_lua_file) 上下文.
* `body_filter`
	可能是 [body_filter_by_lua](http://wiki.nginx.org/HttpLuaModule#body_filter_by_lua) or [body_filter_by_lua_file](http://wiki.nginx.org/HttpLuaModule#body_filter_by_lua_file) 上下文.
* `log`
	可能是 [log_by_lua](http://wiki.nginx.org/HttpLuaModule#log_by_lua) or [log_by_lua_file](http://wiki.nginx.org/HttpLuaModule#log_by_lua_file) 上下文.
* `timer`
	可能是 [ngx.timer.*](http://wiki.nginx.org/HttpLuaModule#ngx.timer.at) 的用户回调函数上下文.

本命令在 `v0.5.10` 版本中引入.

ngx.thread.spawn
----------------
**语法:** *co = ngx.thread.spawn(func, arg1, arg2, ...)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

使用Lua函数`func`及可选参数`arg1`, `arg2`等创建一个新的用户"轻量线程". 返回代表这个*轻量线程*的Lua线程 (或称 Lua coroutine) .

"轻量线程" 其实是一种特殊的由ngx_lua调度的Lua coroutine.

在 `ngx.thread.spawn` 返回之前, `func` 将对那些可选的参数进行调用，直到此函数返回，或因出错而终止，或由于[Nginx API for Lua](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua) (例如[[#tcpsock:receivetcpsock:receive]])的I/O操作而挂起.

`ngx.thread.spawn` 返回之后，新创建的 "轻量线程" 将继续异步地执行，通常会响应不同的I/O事件.

All the Lua code chunks running by [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua), [access_by_lua](http://wiki.nginx.org/HttpLuaModule#access_by_lua), and [content_by_lua](http://wiki.nginx.org/HttpLuaModule#content_by_lua) 运行的所有Lua代码块都运行在一个由ngx_lua自动创建的标准"轻量线程"中. 这个标准"轻量线程"也称为"入口线程".

默认情况下, 只有出现下列情况之一时相应的Nginx处理器 (例如, [rewrite_by_lua](http://wiki.nginx.org/HttpLuaModule#rewrite_by_lua) 处理器)才会终止
1. "入口线程" 和所有用户 "轻量线程" 都结束了,
1. 某个 "轻量线程" (可能是"入口线程" 或某个用户 "轻量线程" 调用了 [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit), [ngx.exec](http://wiki.nginx.org/HttpLuaModule#ngx.exec), [ngx.redirect](http://wiki.nginx.org/HttpLuaModule#ngx.redirect),  [ngx.req.set_uri(uri, true)](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_uri)而终止 
1. "入口线程"由于Lua错误而终止.

但是如果用户 "轻量线程" 由于Lua错误而终止， 它不会象"入口线程"那样导致其它正在运行的 "轻量线程" 被中断.

由于Nginx子请求模型的限制，通常不允许中断一个正在运行的Nginx子请求. 所以也禁止中断一个与一个或多个Nginx子请求绑定的正在运行的 "轻量线程" . 要退出整个"世界"， 你必须调用 [ngx.thread.wait](http://wiki.nginx.org/HttpLuaModule#ngx.thread.wait) 来等待这些 "轻量线程" 结束. 这里有个例外是你可以使用状态码`ngx.ERROR` (-1)，`408`, `444`, or `499`来调用[[#ngx.exitngx.exit]]从而中断某个子请求.

"轻量线程"的调度是抢占式的. 换句话说，分时并不是自动进行的. 一个 "轻量线程" 将独占cpu运行，直到
1. 无法在单独一次运行中完成一个 (非阻塞的) I/O 操作,
1. 它调用 [coroutine.yield](http://wiki.nginx.org/HttpLuaModule#coroutine.yield) 主动放弃执行, 或
1. 由于Lua错误或调用 [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit), [ngx.exec](http://wiki.nginx.org/HttpLuaModule#ngx.exec), [ngx.redirect](http://wiki.nginx.org/HttpLuaModule#ngx.redirect), 或 [ngx.req.set_uri(uri, true)](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_uri)而中断.

对于前两种情况, "轻量线程" 通常会在将来被 ngx_lua 调度器唤醒，除非发生了 "停止所有" 事件.

用户 "轻量线程" 自己也可以创建 "轻量线程" , 由 [[#coroutine.createcoroutine.create]]创建的普通用户coroutine也可以创建"轻量线程". 直接派生出"轻量线程"的这个 coroutine (可能是普通的Lua coroutine 或  "轻量线程") 称为被创建"轻量线程"的 "父coroutine" .

"父coroutine" 可以调用 [ngx.thread.wait](http://wiki.nginx.org/HttpLuaModule#ngx.thread.wait) 来等待子 "轻量线程" 结束.

你可以对"轻量线程"coroutine调用 coroutine.status() 和 coroutine.yield() .

 "轻量线程" coroutine 在下面的情况下可能成为 "僵尸" 
1. 当前 "轻量线程" 已经结束 (成功或者出错),
1. 父corotine仍然活着，而且
1. 父coroutine没有用 [ngx.thread.wait](http://wiki.nginx.org/HttpLuaModule#ngx.thread.wait) 等待它.

下面的例子演示了使用"轻量线程" coroutine.yield() 来实现手动分时:


    local yield = coroutine.yield

    function f()
        local self = coroutine.running()
        ngx.say("f 1")
        yield(self)
        ngx.say("f 2")
        yield(self)
        ngx.say("f 3")
    end

    local self = coroutine.running()
    ngx.say("0")
    yield(self)

    ngx.say("1")
    ngx.thread.spawn(f)

    ngx.say("2")
    yield(self)

    ngx.say("3")
    yield(self)

    ngx.say("4")


将会生成这样的输出


    0
    1
    f 1
    2
    f 2
    3
    f 3
    4


"轻量线程" 对于在一个Nginx请求处理器中实现并发的上游请求最有用, 有些象一个更一般化的可以与所有 [Nginx API for Lua](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)一起使用的 [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi) . 下面的例子演示了在一个Lua处理器中对 MySQL, Memcached, 和上游HTTP 服务的并行请求, 结果按实际返回的次序输出 (非常象Facebook BigPipe 模型):


    -- 同时请求mysql, memcached, 和一个远程http服务,
    -- 结果按实际返回的次序输出
    -- .

    local mysql = require "resty.mysql"
    local memcached = require "resty.memcached"

    local function query_mysql()
        local db = mysql:new()
        db:connect{
                    host = "127.0.0.1",
                    port = 3306,
                    database = "test",
                    user = "monty",
                    password = "mypass"
                  }
        local res, err, errno, sqlstate =
                db:query("select * from cats order by id asc")
        db:set_keepalive(0, 100)
        ngx.say("mysql done: ", cjson.encode(res))
    end

    local function query_memcached()
        local memc = memcached:new()
        memc:connect("127.0.0.1", 11211)
        local res, err = memc:get("some_key")
        ngx.say("memcached done: ", res)
    end

    local function query_http()
        local res = ngx.location.capture("/my-http-proxy")
        ngx.say("http done: ", res.body)
    end

    ngx.thread.spawn(query_mysql)      -- create thread 1
    ngx.thread.spawn(query_memcached)  -- create thread 2
    ngx.thread.spawn(query_http)       -- create thread 3 


本命令在 `v0.7.0` 版本中引入.

ngx.thread.wait
---------------
**语法:** *ok, res1, res2, ... = ngx.thread.wait(thread1, thread2, ...)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

等待一个或多个子 "轻量线程" 并返回第一个结束(成功或出错)的 "轻量线程" 的结果 .

参数`thread1`, `thread2`, 等是由之前[[#ngx.thread.spawnngx.thread.spawn]]调用返回的Lua线程对象.

返回值的意思与 [coroutine.resume](http://wiki.nginx.org/HttpLuaModule#coroutine.resume) 的返回值完全一样, 即, 第一个返回值是表明"轻量线程"是否成功结束的布尔值，后面的值是用来派生出"轻量线程"的Lua函数的返回值(成功的情况)或错误对象（出错的情况）.

只有直接的"父coroutine"可以等待子"轻量线程", 否则会抛出Lua异常.

下面的例子演示了用 `ngx.thread.wait` 和 [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) 来模拟 [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi):


    local capture = ngx.location.capture
    local spawn = ngx.thread.spawn
    local wait = ngx.thread.wait
    local say = ngx.say

    local function fetch(uri)
        return capture(uri)
    end

    local threads = {
        spawn(fetch, "/foo"),
        spawn(fetch, "/bar"),
        spawn(fetch, "/baz")
    }

    for i = 1, #threads do
        local ok, res = wait(threads[i])
        if not ok then
            say(i, ": failed to run: ", res)
        else
            say(i, ": status: ", res.status)
            say(i, ": body: ", res.body)
        end
    end


这里主要实现了 "wait all" 模式.

下面演示 "wait any" 模式:


    function f()
        ngx.sleep(0.2)
        ngx.say("f: hello")
        return "f done"
    end

    function g()
        ngx.sleep(0.1)
        ngx.say("g: hello")
        return "g done"
    end

    local tf, err = ngx.thread.spawn(f)
    if not tf then
        ngx.say("failed to spawn thread f: ", err)
        return
    end

    ngx.say("f thread created: ", coroutine.status(tf))

    local tg, err = ngx.thread.spawn(g)
    if not tg then
        ngx.say("failed to spawn thread g: ", err)
        return
    end

    ngx.say("g thread created: ", coroutine.status(tg))

    ok, res = ngx.thread.wait(tf, tg)
    if not ok then
        ngx.say("failed to wait: ", res)
        return
    end

    ngx.say("res: ", res)

    -- 停止所有，中断当前运行的线程
    ngx.exit(ngx.OK)


输出:


    f thread created: running
    g thread created: running
    g: hello
    res: g done


本命令在 `v0.7.0` 版本中引入.

ngx.on_abort
------------
**语法:** *ok, err = ngx.on_abort(callback)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

将一个用户Lua函数注册成为回调函数，当客户端过早关闭（下游）连接时会调用此回调函数.

注册成功返回`1` 否则返回 `nil` 和错误字符串.

回调函数中可以使用所有的 [Nginx API for Lua](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)，这是因为函数运行于一个特殊的"轻量线程"中，与[[#ngx.thread.spawnngx.thread.spawn]]生成的其它"轻量线程"一样.

此回调函数可以完全自主地决定如何处理客户端中断事件. 例如，它可以简单地忽略掉些事情，不做任何事情，当前的Lua请求处理器将继续执行，不受影响. 回调函数也可以决定用 [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit) 来停止一切, 例如,


    local function my_cleanup()
        -- 这里写自定义的清理工作，如取消一个等待中的数据库事务

        -- 现在中断当前请求处理器中运行的所有"轻量线程"
        ngx.exit(499)
    end

    local ok, err = ngx.on_abort(my_cleanup)
    if not ok then
        ngx.log(ngx.ERR, "failed to register the on_abort callback: ", err)
        ngx.exit(500)
    end


如果[lua_check_client_abort](http://wiki.nginx.org/HttpLuaModule#lua_check_client_abort) 设为 `off` (这是默认值), 则此函数总是返回错误信息 "lua_check_client_abort is off".

按当前的实现，此函数只会在一个请求处理器中被调用一次; 再调则返回错误信息 "duplicate call".

此API在 `v0.7.4` 版本中引入.

参阅[lua_check_client_abort](http://wiki.nginx.org/HttpLuaModule#lua_check_client_abort).

ngx.timer.at
------------
**语法:** *ok, err = ngx.timer.at(delay, callback, user_arg1, user_arg2, ...)*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

使用用户回调函数及可选参数创建Nginx定时器.

第一个参数, `delay`, 指定定时器的延迟,
单位是秒. 可以用小数点如 `0.001` 表示 1
毫秒. 也可以指定`0`延迟，这样当前处理器挂起时定时器将立即过期.

第二个参数, `callback`, 可以是任何Lua函数，在指定的延迟时间后被后台的"轻量线程"调用. 用户回调函数将被Nginx核心自动调用，调用参数为`premature`,
`user_arg1`, `user_arg2`, 等, 其中 `premature`
参数是布尔值，表示是否是过早的定时器过期,  `user_arg1`, `user_arg2`, 等是调用`ngx.timer.at`时指定的那些参数.

过早的定时器过期的发生是`HUP`导致重新加载Nginx配置或Nginx服务器关闭导致Nginx worker进程尝试关闭. 当Nginx worker进程尝试关闭时，不能再调用 `ngx.timer.at` 创建新的定时器，`ngx.timer.at` 将返回`nil` 及错误字符串 "process exiting".

当定时器过期时, 定时器回调函数中的用户Lua代码将在一个与创建定时器的请求脱离的"轻量线程"中运行. 所以，与创建它们的请求具有相同生命周期的对象，如 [cosockets](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp), 不能在原始请求与定时器用户回调函数之间共享.

以下是一个简单的例子:


    location / {
        ...
        log_by_lua '
            local function push_data(premature, uri, args, status)
                -- 使用ngx.socket.tcp 或 ngx.socket.udp将数据uri, 参数和状态推送给远程
                --  
                -- (你可能需要在Lua里对数据进行缓冲以
                -- 减少I/O操作)
            end
            local ok, err = ngx.timer.at(0, push_data,
                                         ngx.var.uri, ngx.var.args, ngx.header.status)
            if not ok then
                ngx.log(ngx.ERR, "failed to create timer: ", err)
                return
            end
        ';
    }


也可以通过在回调函数里递归调用 `ngx.timer.at`创建无限次重复调用的定时器, 例如, 每 `5` 秒触发一次的定时器. 见下例,


    local delay = 5
    local handler
    handler = function (premature)
        -- 用Lua做一些日常任务，就象cron一样
        if premature then
            return
        end
        local ok, err = ngx.timer.at(delay, handler)
        if not ok then
            ngx.log(ngx.ERR, "failed to create the timer: ", err)
            return
        end
    end
    
    local ok, err = ngx.timer.at(delay, handler)
    if not ok then
        ngx.log(ngx.ERR, "failed to create the timer: ", err)
        return
    end


由于定时器回调运行在后台，它们的运行时间并不会增加客户端请求的响应时间，所以它们很容易由于Lua程序错误或太多客户端流量而在服务器上积累，耗尽系统资源. 为了避免象导致Nginx服务器崩溃这种极端后果，对于一个Nginx worker进程中的"等待中"的定时器和"运行中"的定时器数量都有内置的限制. "等待中的定时器" 指尚未到期的定时器 "运行中的定时器" 指其user回调函数当前正在运行的定时器.

Nginx worker中等待中的定时器最大数目由 [lua_max_pending_timers](http://wiki.nginx.org/HttpLuaModule#lua_max_pending_timers)命令控制. 运行中的定时器最大数目由 [lua_max_running_timers](http://wiki.nginx.org/HttpLuaModule#lua_max_running_timers)命令控制.

按照当前的实现，每个 "运行中定时器" 将在`nginx.conf`中用标准[[EventsModule#worker_connectionsworker_connections]]配置的连接记录列表中占据一条（假的）连接记录. 所以要确保 
[worker_connections](http://wiki.nginx.org/EventsModule#worker_connections) 命令设置得足够大, 同时考虑定时器回调函数(由
[[#lua_max_running_timerslua_max_running_timers]] 命令设置)所需要的真实连接和假连接的数量.

在定时器回调函数中可以使用很多供Nginx使用的Lua API, 如流/数据报 cosocket ([ngx.socket.tcp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp) 和 [ngx.socket.udp](http://wiki.nginx.org/HttpLuaModule#ngx.socket.udp)), 共享内存字典 ([ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT)), 用户coroutine ([coroutine.*](http://wiki.nginx.org/HttpLuaModule#coroutine.create)),
用户 "轻量线程" ([ngx.thread.*](http://wiki.nginx.org/HttpLuaModule#ngx.thread.spawn)), [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit), [ngx.now](http://wiki.nginx.org/HttpLuaModule#ngx.now)/[ngx.time](http://wiki.nginx.org/HttpLuaModule#ngx.time),
[ngx.md5](http://wiki.nginx.org/HttpLuaModule#ngx.md5)/[ngx.sha1_bin](http://wiki.nginx.org/HttpLuaModule#ngx.sha1_bin), 都可以使用. 但是子请求API (如[ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture)), [ngx.req.*](http://wiki.nginx.org/HttpLuaModule#ngx.req.start_time) API, 下游输出 API
(如 [ngx.say](http://wiki.nginx.org/HttpLuaModule#ngx.say), [ngx.print](http://wiki.nginx.org/HttpLuaModule#ngx.print), 和[ngx.flush](http://wiki.nginx.org/HttpLuaModule#ngx.flush)) 在这个上下文中都是明确禁止使用的.

此API在 `v0.8.0` 版本中引入.

ndk.set_var.DIRECTIVE
---------------------
**语法:** *res = ndk.set_var.DIRECTIVE_NAME*

**可包含于:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

此机制允许调用由[Nginx Devel Kit](https://github.com/simpl/ngx_devel_kit) (NDK) set_var 子模块的 `ndk_set_var_value`实现的其它nginx C模块命令。.

For example, the following [HttpSetMiscModule](http://wiki.nginx.org/HttpSetMiscModule) directives can be invoked this way:

* [set_quote_sql_str](http://wiki.nginx.org/HttpSetMiscModule#set_quote_sql_str)
* [set_quote_pgsql_str](http://wiki.nginx.org/HttpSetMiscModule#set_quote_pgsql_str)
* [set_quote_json_str](http://wiki.nginx.org/HttpSetMiscModule#set_quote_json_str)
* [set_unescape_uri](http://wiki.nginx.org/HttpSetMiscModule#set_unescape_uri)
* [set_escape_uri](http://wiki.nginx.org/HttpSetMiscModule#set_escape_uri)
* [set_encode_base32](http://wiki.nginx.org/HttpSetMiscModule#set_encode_base32)
* [set_decode_base32](http://wiki.nginx.org/HttpSetMiscModule#set_decode_base32)
* [set_encode_base64](http://wiki.nginx.org/HttpSetMiscModule#set_encode_base64)
* [set_decode_base64](http://wiki.nginx.org/HttpSetMiscModule#set_decode_base64)
* [set_encode_hex](http://wiki.nginx.org/HttpSetMiscModule#set_encode_base64)
* [set_decode_hex](http://wiki.nginx.org/HttpSetMiscModule#set_decode_base64)
* [set_sha1](http://wiki.nginx.org/HttpSetMiscModule#set_encode_base64)
* [set_md5](http://wiki.nginx.org/HttpSetMiscModule#set_decode_base64)

例如,


    local res = ndk.set_var.set_escape_uri('a/b');
    -- 现在 res == 'a%2fb'


类似的, 也可以在Lua中调用 [HttpEncryptedSessionModule](http://wiki.nginx.org/HttpEncryptedSessionModule)提供的命令:

* [set_encrypt_session](http://wiki.nginx.org/HttpEncryptedSessionModule#set_encrypt_session)
* [set_decrypt_session](http://wiki.nginx.org/HttpEncryptedSessionModule#set_decrypt_session)

这条功能需要 [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit) 模块.

coroutine.create
----------------
**语法:** *co = coroutine.create(f)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

用Lua函数创建一个用户Lua coroutine，返回coroutine对象.

与标准的Lua [coroutine.create](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.create) API类似, 但工作在由ngx_lua创建的Lua coroutine上下文中.

此API在 `v0.6.0` 版本中引入.

coroutine.resume
----------------
**语法:** *ok, ... = coroutine.resume(co, ...)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

恢复一个之前挂起或刚创建的用户Lua coroutine的执行.

与标准的Lua [coroutine.resume](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.resume) API类似, 但工作在由ngx_lua创建的Lua coroutine上下文中.

此API在 `v0.6.0` 版本中引入.

coroutine.yield
---------------
**语法:** *... = coroutine.yield(co, ...)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

挂起当前Lua coroutine的执行.

与标准的Lua [coroutine.yield](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.yield) API类似, 但工作在由ngx_lua创建的Lua coroutine上下文中.

此API在 `v0.6.0` 版本中引入.

coroutine.wrap
--------------
**语法:** *co = coroutine.wrap(f)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

与标准的Lua [coroutine.wrap](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.wrap) API类似, 但工作在由ngx_lua创建的Lua coroutine上下文中.

此API在 `v0.6.0` 版本中引入.

coroutine.running
-----------------
**语法:** *co = coroutine.running()*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

与标准 Lua [coroutine.running](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.running) API相同.

此API在 `v0.6.0` 版本中引入.

coroutine.status
----------------
**语法:** *status = coroutine.status(co)*

**可包含于:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

与标准 Lua [coroutine.status](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.status) API相同.

此API在 `v0.6.0` 版本中引入.

Lua/LuaJIT 字节码支持
==========================

从 `v0.5.0rc32` 版本开始, 所有 `*_by_lua_file` 配置命令(如 [content_by_lua_file](http://wiki.nginx.org/HttpLuaModule#content_by_lua_file)) 支持直接加载Lua 5.1 和 LuaJIT 2.0 原始字节码文件.

请注意 LuaJIT 2.0 使用的字节码格式与标准Lua 5.1解释器使用的格式不兼容. 所以如果在ngx_lua中使用 LuaJIT 2.0, 需要这样生成LuaJIT兼容的字节码:


    /path/to/luajit/bin/luajit -b /path/to/input_file.lua /path/to/output_file.luac


可以用 `-bg` 选项在LuaJIT字节码文件中包含调试信息:


    /path/to/luajit/bin/luajit -bg /path/to/input_file.lua /path/to/output_file.luac


请参阅官方 LuaJIT 文档了解 `-b` 选项的细节:

<http://luajit.org/running.html#opt_b>

类似的, 如果在ngx_lua中使用标准Lua 5.1, 要象下面这样用`luac`生成兼容的字节码文件:


    luac -o /path/to/output_file.luac /path/to/input_file.lua


与LuaJIT不同, 标准 Lua 5.1 字节码文件中默认就携带调试信息. 可以用 `-s` 选项去掉调试信息:


    luac -s -o /path/to/output_file.luac /path/to/input_file.lua


试图在连接了LuaJIT2.0的ngx_lua实例中加载标准Lua5.1的字节码文件，或相反，都将导致以下错误信息被记入nginx `error.log` 文件:


    [error] 13909#0: *1 failed to load Lua inlined code: bad byte-code header in /path/to/test_file.luac


使用Lua内置的操作如 `require` 和 `dofile` 加载字节码文件应该总能工作.

HTTP 1.0 支持
===============

HTTP 1.0 协议不支持分块输出， 要支持HTTP 1.0 keep-alive，在响应体非空时，在响应头中需要显式地提供`Content-Length`.
所以如果是HTTP 1.0请求而且[lua_http10_buffering](http://wiki.nginx.org/HttpLuaModule#lua_http10_buffering) 命令被置为 `on`, ngx_lua 将会缓存把 [ngx.say](http://wiki.nginx.org/HttpLuaModule#ngx.say) 和[ngx.print](http://wiki.nginx.org/HttpLuaModule#ngx.print) 的输出，并延迟发送响应头，直到得到所有的响应体.
这时 ngx_lua 就可以计算出体的总长度从而创建正确的 `Content-Length` 头返回给 HTTP 1.0 客户端.
但是如果在运行的Lua代码中设置了 `Content-Length` 响应头, 即使[lua_http10_buffering](http://wiki.nginx.org/HttpLuaModule#lua_http10_buffering)命令被置为 `on` 也不会做这个缓存.

对于大的流式输出响应，要记住禁止[[#lua_http10_bufferinglua_http10_buffering]]以减少内存占用.

注意常用的HTTP benchmark工具如 `ab` 和 `http_load` 默认发送 HTTP 1.0 请求.
要强制 `curl` 发送 HTTP 1.0 请求, 使用 `-0` 选项.

在一个Nginx Worker进程内部共享数据
=============================================

要在同一个nginx worker进程处理的所有请求之间共享数据，将共享数据封装到一个Lua模块中，使用内置的 `require` 引入模块, 然后用Lua操作共享的数据. 其工作原理是被require的Lua模块仅加载一次，所有的coroutine将共享这个模块的同一个拷贝 (包括代码和数据). 注意，相反， Lua 全局变量 (注意，不是模块级别的变量) 将不会跨请求，这是由一个请求对应一个coroutine的设计理念决定的.

下面是一个完整的例子:


    -- mydata.lua
    module(...)
 
    local data = {
        dog = 3,
        cat = 4,
        pig = 5,
    }
 
    function get_age(name)
        return data[name]
    end


然后在 `nginx.conf` 中访问它:


    location /lua {
        content_by_lua '
            local mydata = require "mydata"
            ngx.say(mydata.get_age("dog"))
        ';
    }


本例中的 `mydata` 模块代码仅在第一个对`/lua`的请求中加载，然后运行, 所有对同一个nginx worker进程的后续请求将使用这个模块的同一个实例，包括其中的数据，直到向Nginx master进程发送 `HUP` 信号强制重新加载配置.
这种数据共享技术对于使用本模块的高性能Lua应用是至关重要的.

注意这种数据共享是 *worker*级别 而不是 *server*级别 的. 也就是说，如果在Nginx master下有多个nginx worker进程，数据共享不会跨越这些worker之间的进程边界.

如果真需要实现在整个server级别的数据共享, 使用以下方法之一:
1. 使用本模块提供的 [ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT) API.
1. 仅使用一个nginx worker和一个 server (在多核或多CPU的机器上不推荐这么做).
1. 使用 `memcached`, `redis`, `MySQL` 或r `PostgreSQL` 之类的存储机制. 与本模块紧密联系的[The ngx_openresty 包](http://openresty.org) 带有一系列 Nginx modules 和 Lua 库可以提供与这些数据存储机制交互的方法.

已知问题
============

TCP socket 连接操作问题
-----------------------------
[tcpsock:connect](http://wiki.nginx.org/HttpLuaModule#tcpsock:connect) 方法可能在连接失败如`Connection Refused`时返回 `success` . 

但是，后续对cosocket对象的操作将失败，返回由连接操作失败生成的错误状态信息. 

此问题源于Nginx事件模型的限制，似乎仅在Mac OS X上发生.

Lua Coroutine 的挂起/恢复
------------------------------
* Lua内置的 `dofile` 在Lua 5.1和LuaJIT2.0中都是用 C 函数实现的， 如果在将被`dofile`加载的文件中调用了[ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture), [ngx.exec](http://wiki.nginx.org/HttpLuaModule#ngx.exec), [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit) or [ngx.req.read_body](http://wiki.nginx.org/HttpLuaModule#ngx.req.read_body) , 将导致跨越C函数边界的coroutine挂起. 这在 ngx_lua 中是不允许的，将会导致错误信息如 `lua handler aborted: runtime error: attempt to yield across C-call boundary`. 为避免这种错误，定义一个真正的 Lua 模块而用内置的 `require` 加载.
* 由于标准的Lua 5.1 解释器虚拟机不是完全可恢复的, 如果使用标准的Lua 5.1 解释器， [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture), [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi), [ngx.redirect](http://wiki.nginx.org/HttpLuaModule#ngx.redirect), [ngx.exec](http://wiki.nginx.org/HttpLuaModule#ngx.exec), 和 [ngx.exit](http://wiki.nginx.org/HttpLuaModule#ngx.exit) 不能用于 Lua [pcall()](http://www.lua.org/manual/5.1/manual.html#pdf-pcall) 或 [xpcall()](http://www.lua.org/manual/5.1/manual.html#pdf-xpcall) 的上下文甚至是 `for ... in ...` 语句的第一行，会产生 `attempt to yield across metamethod/C-call boundary` 错误. 请使用 LuaJIT 2.0, 它支持完全可恢复虚拟机, 来避免这个问题.

Lua 变量作用域
-------------------
在导入模块时需要小心，应该使用这种形式:


    local xxx = require('xxx')


	而不是已经不推荐的形式:


    require('xxx')


原因是: 在设计上，全局环境拥有与其关联的Nginx请求处理器完全一样的生命周期. 每个请求处理器都有它自己的Lua全局变量集，这就是请求隔离的概念. Lua模块是由第一个Nginx请求处理器加载的，由内置的`require()`缓存在package.loaded table中，供后续使用, `require()` 有一个副作用，会在加载的模块table中设置一个全局变量. 但是这个全局变量会在请求处理器结束时被清除，每个后续的请求处理器都有自己的（干净的）全局环境. 这样就会因为获取不到模块导致Lua异常.

建议总是在使用I/O操作的Lua模块的最后加上下面这段代码以防止随意使用由*所有*请求共享的模块级全局变量:


    local class_mt = {
        -- 防止随意使用模块全局变量
        __newindex = function (table, key, val)
            error('attempt to write to undeclared variable "' .. key .. '"')
        end
    }
    setmetatable(_M, class_mt)


这全保证Lua模块函数中的本地变量都被加上 `local` 关键字, 否则就会抛出运行时异常. 这避免了访问这些变量时的不希望发生的竞争条件. 参阅 [Data Sharing within an Nginx Worker](http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker) 了解背后的原因.

由子请求命令或其它模块配置的Location
--------------------------------------------------
[ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) 和 [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi) 命令不能获取包含 [echo_location](http://wiki.nginx.org/HttpEchoModule#echo_location), [echo_location_async](http://wiki.nginx.org/HttpEchoModule#echo_location_async), [echo_subrequest](http://wiki.nginx.org/HttpEchoModule#echo_subrequest), 或 [echo_subrequest_async](http://wiki.nginx.org/HttpEchoModule#echo_subrequest_async) 命令的location.


    location /foo {
        content_by_lua '
            res = ngx.location.capture("/bar")
        ';
    }
    location /bar {
        echo_location /blah;
    }
    location /blah {
        echo "Success!";
    }



    $ curl -i http://example.com/foo


以上代码无法正常运行.

特殊PCRE序列
----------------
PCRE sequences such as `\d`, `\s`, or `\w`这样的PCRE序列需要引起特别的重视，因为在字符串写法里， `\`会被Lua语言解析器和Nginx配置文件解析器在开始处理前删掉. 所以下面的代码无法正常运行:


    # nginx.conf
    ?location /test {
    ?content_by_lua '
    ?local regex = "\d+"  -- 这是错误的!!    ?local m = ngx.re.match("hello, 1234", regex)
    ?if m then ngx.say(m[0]) else ngx.say("not matched!") end
    ?';
    ?}
    # evaluates to "not matched!"


为了避免错误，要进行 *双重* 转义 :


    # nginx.conf
    location /test {
        content_by_lua '
            local regex = "\\\\d+"
            local m = ngx.re.match("hello, 1234", regex)
            if m then ngx.say(m[0]) else ngx.say("not matched!") end
        ';
    }
    # 值为 "1234"


这里, `\\\\d+` 被Nginx配置文件解析器处理成 `\\d+` ，然后在运行之前被Lua语言解析器处理成 `\d+` .

或者可以将正则表达式写在双方括号里, `[[...]]`, 这样反斜线只会被Nginx配置文件处理器转义一次. 


    # nginx.conf
    location /test {
        content_by_lua '
            local regex = [[\\d+]]
            local m = ngx.re.match("hello, 1234", regex)
            if m then ngx.say(m[0]) else ngx.say("not matched!") end
        ';
    }
    # 值为 "1234"


这里, `[[\\d+]]` 被Nginx配置文件解析器处理成 `[[\d+]]` 可以正常执行.

注意当正则表达式包括`[...]`序列时，可能需要更长的方括号形式`[=[...]=]`. 
如果需要可以将 `[=[...]=]` 作为默认的正则书写形式.


    # nginx.conf
    location /test {
        content_by_lua '
            local regex = [=[[0-9]+]=]
            local m = ngx.re.match("hello, 1234", regex)
            if m then ngx.say(m[0]) else ngx.say("not matched!") end
        ';
    }
    # 值为 "1234"


另外一种对PCRE序列进行转义的方法是确保Lua代码放在外部脚本文件里，通过各种 `*_by_lua_file` 命令来调用. 
使用这种方法，反斜线仅会被Lua语言解析器处理一次，因此不必多次转义.


    -- test.lua
    local regex = "\\d+"
    local m = ngx.re.match("hello, 1234", regex)
    if m then ngx.say(m[0]) else ngx.say("not matched!") end
    -- evaluates to "1234"


在外部脚本文件里，用双方括号表示的Lua字符串不需要作修改。. 
 

    -- test.lua
    local regex = [[\d+]]
    local m = ngx.re.match("hello, 1234", regex)
    if m then ngx.say(m[0]) else ngx.say("not matched!") end
    -- evaluates to "1234"


典型用法
============

以下列出部分:

* 在Lua中对各个nginx上游(proxy, drizzle, postgres, redis, memcached等)输出进行混合和处理,
* 在请求真正到达上游后端前用Lua进行复杂的访问控制和安全检测,
* (使用Lua)对响应头进行任意操作
* 从后端外部存储(象 redis, memcached, mysql, postgresql)获取数据，用这些数据来实时选择访问哪个上游后端,
* 在内容处理器中用同步但仍是非阻塞的方式访问后端数据库或其它存储来实现任意复杂的web应用,
* 在rewrite阶段用Lua完成非常复杂的URL派发,
* 使用Lua为Nginx子请求和任何location实现高级缓存机制.

本模块允许将Nginx内部的不同元素放到一起并且将Lua语言的强大赋予了用户，拥有无限的可能性. 本模块提供了脚本语言的全部灵活性，以及无论在CPU时间上还是在内存消耗上都与C语言程序相匹敌的高性能. 在使用LuaJit 2.0时尤其如此. 

其它的脚本语言实现一般都很难达到这个级别的性能.

Lua状态 (Lua虚拟机实例) 在同一个nginx worker进程中被所有请求共享以减少内存占用.

在一个 ThinkPad T400 2.80 GHz 笔记本上, 使用 `http_load -p 10`， Hello World例子轻松达到 28k req/sec . 相比之下, Nginx + php-fpm 5.2.8 + Unix Domain Socket 的hello work结果为 6k req/sec ， [Node.js](http://nodejs.org/) v0.6.1 的hello world结果为 10.2k req/sec .

Nginx 兼容性
===============
最新的模块版本兼容以下Nginx版本:

* 1.4.x (已测试: 1.4.1)
* 1.3.x (已测试 : 1.3.11)
* 1.2.x (已测试: 1.2.9)
* 1.1.x (已测试: 1.1.5)
* 1.0.x (已测试: 1.0.15)
* 0.9.x (已测试: 0.9.4)
* 0.8.x >= 0.8.54 (已测试: 0.8.54)

代码仓库
============

本项目的代码托管在github [chaoslawful/lua-nginx-module](http://github.com/chaoslawful/lua-nginx-module).

安装
======

[ngx_openresty 包](http://openresty.org) 可以用来安装Nginx, ngx_lua, 标准 Lua 5.1 解释器或 LuaJIT 2.0, 以及一堆强大的Nginx模块. 基本安装过程是 `./configure --with-luajit && make && make install`.

也可以手动将ngx_lua编译进Nginx:

1. 安装 LuaJIT 2.0 (推荐) 或 Lua 5.1 (Lua 5.2 *尚* 不支持). LuajIT 可以从 [LuaJIT 官网](http://luajit.org/download.html)下载， Lua 5.1 从 [Lua官网](http://www.lua.org/)下载.  有一些发行版包管理器也发布 LuajIT 和/或 Lua.
1. 从 [这里](http://github.com/simpl/ngx_devel_kit/tags) 下载ngx_devel_kit (NDK) 模块的最新版本.
1. 从 [这里](http://github.com/chaoslawful/lua-nginx-module/tags)下载ngx_lua的最新版本.
1. 从 [这里](http://nginx.org/)下载Nginx的最新版本 (见[Nginx兼容性](http://wiki.nginx.org/HttpLuaModule#Nginx_Compatibility))

将本模块编译进Nginx:


    wget 'http://nginx.org/download/nginx-1.4.1.tar.gz'
    tar -xzvf nginx-1.4.1.tar.gz
    cd nginx-1.4.1/

    # 告诉Nginx编译系统在哪里找到 LuaJIT:
    export LUAJIT_LIB=/path/to/luajit/lib
    export LUAJIT_INC=/path/to/luajit/include/luajit-2.0
 
    # 如果使用Lua就告诉它在哪里找到Lua:
    #export LUA_LIB=/path/to/lua/lib
    #export LUA_INC=/path/to/lua/include
 
    # 这里我们假设Nginx安装到 /opt/nginx/.
    ./configure --prefix=/opt/nginx \
            --add-module=/path/to/ngx_devel_kit \
            --add-module=/path/to/lua-nginx-module
 
    make -j2
    make install


在 Ubuntu 11.10 上安装
--------------------------

注意，推荐尽可能使用 LuaJIT 2.0 而不是标准 Lua 5.1 解释器. 

如果一定要使用标准Lua 5.1 解释器, 运行以下命令从Ubuntu仓库安装:


    apt-get install -y lua5.1 liblua5.1-0 liblua5.1-0-dev


一切都应该顺利安装，只有一个小问题. 

liblua5.1包中的库名 `liblua.so` 改了，成了 `liblua5.1.so`, 需要以 `/usr/lib` 中添加符号链接使得在配置过程中能找到这个库.


    ln -s /usr/lib/x86_64-linux-gnu/liblua5.1.so /usr/lib/liblua.so


社区
======

英文邮件列表
------------------

[openresty-en](https://groups.google.com/group/openresty-en) 

中文邮件列表
------------------

[openresty](https://groups.google.com/group/openresty).

Bug与补丁
============

请用以下方法提交bug，功能需求或补丁

1. 提交bug到 [GitHub Issue Tracker](http://github.com/chaoslawful/lua-nginx-module/issues),
1. 或发邮件到 [OpenResty community](http://wiki.nginx.org/HttpLuaModule#Community).

TODO
====

Short Term
----------
* review and apply Brian Akin's patch for the new directive `lua_socket_log_errors`.
* review and apply Brian Akin's patch for the new `shdict:flush_expired()` API.
* implement the SSL cosocket API.
* review and apply Jader H. Silva's patch for `ngx.re.split()`.
* review and apply vadim-pavlov's patch for [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture)'s `extra_headers` option
* use `ngx_hash_t` to optimize the built-in header look-up process for [ngx.req.set_header](http://wiki.nginx.org/HttpLuaModule#ngx.req.set_header), [ngx.header.HEADER](http://wiki.nginx.org/HttpLuaModule#ngx.header.HEADER), and etc.
* add configure options for different strategies of handling the cosocket connection exceeding in the pools.
* add directives to run Lua codes when nginx stops.
* add APIs to access cookies as key/value pairs.
* add `ignore_resp_headers`, `ignore_resp_body`, and `ignore_resp` options to [ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture) and [ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi) methods, to allow micro performance tuning on the user side.
* implement new directive `lua_ignore_client_abort`.

Longer Term
-----------
* add lightweight thread API (i.e., the `ngx.thread` API) as demonstrated in [this sample code](http://agentzh.org/misc/nginx/lua-thread2.lua).
* add automatic Lua code time slicing support by yielding and resuming the Lua VM actively via Lua's debug hooks.
* add `stat` mode similar to [mod_lua](http://httpd.apache.org/docs/2.3/mod/mod_lua.html).

变动
======

本模块的每次版本发布的变动信息可以从 ngx_openresty 发布包中的变动日志中获得:

<http://openresty.org/#Changes>

测试包
=========

要运行测试包需要以下依赖:

* Nginx 版本>= 0.8.54

* Perl 模块:
	* test-nginx: <http://github.com/agentzh/test-nginx> 

* Nginx 模块:
	* echo-nginx-module: <http://github.com/agentzh/echo-nginx-module> 
	* drizzle-nginx-module: <http://github.com/chaoslawful/drizzle-nginx-module> 
	* rds-json-nginx-module: <http://github.com/agentzh/rds-json-nginx-module> 
	* set-misc-nginx-module: <http://github.com/agentzh/set-misc-nginx-module> 
	* headers-more-nginx-module: <http://github.com/agentzh/headers-more-nginx-module> 
	* memc-nginx-module: <http://github.com/agentzh/memc-nginx-module> 
	* srcache-nginx-module: <http://github.com/agentzh/srcache-nginx-module> 
	* ngx_auth_request: <http://mdounin.ru/hg/ngx_http_auth_request_module/> 

* C 库:
	* yajl: <https://github.com/lloyd/yajl> 

* Lua 模块:
	* lua-yajl: <https://github.com/brimworks/lua-yajl> 
		* Note: the compiled module has to be placed in '/usr/local/lib/lua/5.1/'

* 应用程序:
	* mysql: create database 'ngx_test', grant all privileges to user 'ngx_test', password is 'ngx_test'
	* memcached

这些模块在配置中的添加次序是重要的，因为过滤器链中任何过滤器模块的位置都将对最终输出产生影响. 正确的加载次序是:

1. ngx_devel_kit
1. set-misc-nginx-module
1. ngx_http_auth_request_module
1. echo-nginx-module
1. memc-nginx-module
1. lua-nginx-module (i.e. this module)
1. headers-more-nginx-module
1. srcache-nginx-module
1. drizzle-nginx-module
1. rds-json-nginx-module

Copyright and License
=====================

This module is licensed under the BSD license.

Copyright (C) 2009-2013, by Xiaozhe Wang (chaoslawful) <chaoslawful@gmail.com>.

Copyright (C) 2009-2013, by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

参阅
======

* [lua-resty-memcached](http://github.com/agentzh/lua-resty-memcached) library based on ngx_lua cosocket.
* [lua-resty-redis](http://github.com/agentzh/lua-resty-redis) library based on ngx_lua cosocket.
* [lua-resty-mysql](http://github.com/agentzh/lua-resty-mysql) library based on ngx_lua cosocket.
* [lua-resty-upload](http://github.com/agentzh/lua-resty-upload) library based on ngx_lua cosocket.
* [lua-resty-dns](http://github.com/agentzh/lua-resty-dns) library based on ngx_lua cosocket.
* [lua-resty-string](http://github.com/agentzh/lua-resty-string) library based on [LuaJIT FFI](http://luajit.org/ext_ffi.html).
* [Routing requests to different MySQL queries based on URI arguments](http://openresty.org/#RoutingMySQLQueriesBasedOnURIArgs)
* [Dynamic Routing Based on Redis and Lua](http://openresty.org/#DynamicRoutingBasedOnRedis)
* [Using LuaRocks with ngx_lua](http://openresty.org/#UsingLuaRocks)
* [Introduction to ngx_lua](https://github.com/chaoslawful/lua-nginx-module/wiki/Introduction)
* [ngx_devel_kit](http://github.com/simpl/ngx_devel_kit)
* [HttpEchoModule](http://wiki.nginx.org/HttpEchoModule)
* [HttpDrizzleModule](http://wiki.nginx.org/HttpDrizzleModule)
* [postgres-nginx-module](http://github.com/FRiCKLE/ngx_postgres)
* [HttpMemcModule](http://wiki.nginx.org/HttpMemcModule)
* [The ngx_openresty bundle](http://openresty.org)
* [Nginx Systemtap Toolkit](https://github.com/agentzh/nginx-systemtap-toolkit)

