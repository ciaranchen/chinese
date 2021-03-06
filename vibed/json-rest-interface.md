# JSON REST Interface

Vibe.d allows to quickly implement a JSON webservice.
If we want to implement the following JSON output for
a HTTP request to `/api/v1/chapters`:
Vibe.d允许快速实现JSON 网络服务。
如果我们想为`/api/v1/chapters`的HTTP请求实现以下JSON输出：

    [
      {
        "title": "Hello",
        "id": 1,
        "sections": [
          {
            "title": "World",
            "id": 1
          }
        ]
      },
      {
        "title": "Advanced",
        "id": 2,
        "sections": []
      }
    ]

First define an interface that implements the
according functions and D `struct`s that are
serialized **1:1**:
首先定义一个实现相应函数和D中**1:1**序列化`结构体(struct)`的接口：

    interface IRest
    {
        struct Section {
            string title;
            int id;
        }
        struct Chapter {
            string title;
            int id;
            Section[] sections;
        }
        @path("/api/v1/chapters")
        Chapter[] getChapters();
    }

To actual fill the data structures, we have to inherit
from the interface and implement the business logic:
为了确实填充数据结构，我们必须从接口继承并实现业务逻辑：

    class Rest: IRest {
        Chapter[] getChapters() {
          // fill
        }
    }

Given an `URLRouter` instance we register
an instance of the `Rest` class and we're done!
对于一个`URLRouter`实例，我们注册一个`Rest`类的实例，我们就完成了！

    auto router = new URLRouter;
    router.registerRestInterface(new Rest);

Vibe.d *REST interface generator* also supports
POST request where the children of the posted
JSON object are mapped to the member function's
parameters.
Vibe.d的*REST接口生成器*还支持发送JSON对象的属性映射到成员函数的参数的POST请求。

The REST interface can be used to generate a REST
client instance which transparently sends JSON requests
to the given server, using the same member functions
used on the backend side. This is useful when code
is shared between client and server.
REST接口可用于生成REST客户端实例，该实例通过在后端使用的相同成员函数将JSON请求透明地发送给给定的服务器。
当代码在客户端和服务器之间共享时这很有用。

    auto api = new RestInterfaceClient!IRest("http://127.0.0.1:8080/");
    // sends GET /api/v1/chapters and deserializes
    // response to IRest.Chapter[]
    auto chapters = api.getChapters();

## {SourceCode:disabled}

```d
import vibe.d;

interface IRest
{
    struct Section
    {
        string title;
        int id;
    }

    struct Chapter
    {
        string title;
        int id;
        Section[] sections;
    }

    @path("/api/v1/chapters")
    Chapter[] getChapters();

    /*
    Post data as:
        { "title": "D Language" }
    */
    @path("/api/v1/add-chapter")
    @method(HTTPMethod.POST)
    int addChapter(string title);
}

class Rest: IRest
{
    private Chapter[] chapters_;

    this()
    {
        chapters_ = [ Chapter("Hello", 1,
                [ Section("World", 1) ] ),
                 Chapter("Advanced", 2) ];
    }

    Chapter[] getChapters()
    {
        return chapters_;
    }

    int addChapter(string title)
    {
        import std.algorithm : map, max, reduce;
        // Generate the next highest ID
        auto newId = chapters_.map!(x => x.id)
                            .reduce!max + 1;
        chapters_ ~= Chapter(title, newId);
        return newId;
    }
}

shared static this()
{
    auto router = new URLRouter;
    router.registerRestInterface(new Rest);

    auto settings = new HTTPServerSettings;
    settings.port = 8080;
    listenHTTP(settings, router);
}
```
