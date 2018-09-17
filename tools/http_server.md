## http_server.py

### import
``` py
#!/usr/bin/env python
# Many tests expect there to be an http server on port 4545 servering the deno
# root directory.
import os
from threading import Thread
import SimpleHTTPServer
import SocketServer
from util import root_path
from time import sleep

PORT = 4545
```

### serve_forever

运行测试服务器

```py
def serve_forever():
    os.chdir(root_path)  # 希望主线程也不是 chdir.
    Handler = SimpleHTTPServer.SimpleHTTPRequestHandler
    SocketServer.TCPServer.allow_reuse_address = True
    httpd = SocketServer.TCPServer(("", PORT), Handler)
    print "Deno test server http://localhost:%d/" % PORT
    httpd.serve_forever()
```

- [x] [root_path](./util.md#import)

### spawn

新的守护进程

```py
def spawn():
    thread = Thread(target=serve_forever)
    thread.daemon = True
    thread.start()
    sleep(1)  # TODO 我懒得弄清楚如何正确地做到这一点。
    return thread

```

### __main__

``` py
if __name__ == '__main__':
    serve_forever()
```