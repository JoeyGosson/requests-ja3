# Python3 绕过Ja3验证

[TOC]

### 1. 使用方法

- requests 是基于 urllib3 实现的。要修改 JA3 相关的底层参数，所以我们今天要修改 urllib3 里面的东西。

-  JA3 指纹里面，很大的一块就是 Cipher Suits，也就是加密算法。而 requests 里面默认的加密算法如下：

  ```json
  ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+HIGH:DH+HIGH:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+HIGH:RSA+3DES:!aNULL:!eNULL:!MD5
  ```

- 冒号分割了不同的加密算法。这些加密算法每一种顺序其实就对应了一个 JA3 指纹字符串，只要我们修改这个顺序，就能得到不同的JA3字符串.

- 在 requests 里面，要修改 Cipher Suits 中的加密算法，需要修改 urllib3 里面的 ssl  上下文，并实现一个新的 HTTP 适配器 (HTTPAdapter)。在这个适配器里面，我们在每次请求的时候，随机更换加密算法。但需要注意的是`!aNULL:!eNULL:!MD5`就不用修改了，让他们保持在最后。

- 涉及到的代码如下：

  ```python
  from requests.adapters import HTTPAdapter
  from requests.packages.urllib3.util.ssl_ import create_urllib3_context
  
  ORIGIN_CIPHERS = ('ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+HIGH:'
  'DH+HIGH:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+HIGH:RSA+3DES')
  
  
  class DESAdapter(HTTPAdapter):
      def __init__(self, *args, **kwargs):
          """
          A TransportAdapter that re-enables 3DES support in Requests.
          """
          CIPHERS = ORIGIN_CIPHERS.split(':')
          random.shuffle(CIPHERS)
          CIPHERS = ':'.join(CIPHERS)
          self.CIPHERS = CIPHERS + ':!aNULL:!eNULL:!MD5'
          super().__init__(*args, **kwargs)
          
          
      def init_poolmanager(self, *args, **kwargs):
          context = create_urllib3_context(ciphers=self.CIPHERS)
          kwargs['ssl_context'] = context
          return super(DESAdapter, self).init_poolmanager(*args, **kwargs)
  
      def proxy_manager_for(self, *args, **kwargs):
          context = create_urllib3_context(ciphers=self.CIPHERS)
          kwargs['ssl_context'] = context
          return super(DESAdapter, self).proxy_manager_for(*args, **kwargs)
  ```

- 其中，`s.mount`的第一个参数表示这个适配器只在`https://ja3er.com`开头的网址中生效。

  