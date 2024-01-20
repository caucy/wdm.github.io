# nginx 在tls 的时候如何判断是走h1, 还是h2?

## 背景
一个业务找到我，他们用自研框架，grpc问连接443 的时候，为什么nginx 总返回400，而且nginx 没有任何日志。我初步判断，应该是没走到http阶段就出错了，照理说http 阶段一定会有日志才对。

## 问题复现
### 1，先看为什么400
根据断点显示，请求跑到ngx_http_parse_request_line，解析schema 的时候出错，也就是 POST /uri http1.1，解析这一行出错。具体在[schema 解析](https://github.com/nginx/nginx/blob/master/src/http/ngx_http_parse.c#L104)
，这里h2 的header是PRI * http2.0， 是不会走到ngx_http_parse_request_line 的，所以应该是没成功走到h2，错误的走到了h1，然后报协议错误。
```
PRI *m HTTP/2.0\r\n\r\nSM\r\n\r\n
```


### 2，定位为什么通用grpc 框架可以正常访问
  ssl 握手的时候，会注册h1, h2的回调，定位下为什么h2 回调没设置成功。握手到选协议的过程大概是：
  ngx_http_init_connection
    ->ngx_http_ssl_handshake
      ->ngx_http_ssl_handshake_handler
  在ngx_http_ssl_handshake_handler 选择握手后走h1, 还是h2。
  ```
  ngx_http_ssl_handshake_handler(ngx_connection_t *c)
{
    if (c->ssl->handshaked) {

        /*
         * The majority of browsers do not send the "close notify" alert.
         * Among them are MSIE, old Mozilla, Netscape 4, Konqueror,
         * and Links.  And what is more, MSIE ignores the server's alert.
         *
         * Opera and recent Mozilla send the alert.
         */

        c->ssl->no_wait_shutdown = 1;

#if (NGX_HTTP_V2                                                              \
     && defined TLSEXT_TYPE_application_layer_protocol_negotiation)
        {
        unsigned int             len;
        const unsigned char     *data;
        ngx_http_connection_t   *hc;
        ngx_http_v2_srv_conf_t  *h2scf;

        hc = c->data;

        h2scf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_v2_module);

        if (h2scf->enable || hc->addr_conf->http2) {

            SSL_get0_alpn_selected(c->ssl->connection, &data, &len);

            if (len == 2 && data[0] == 'h' && data[1] == '2') {
                ngx_http_v2_init(c->read);
                return;
            }
        }
        }
#endif

        c->log->action = "waiting for request";

        c->read->handler = ngx_http_wait_request_handler;
        /* STUB: epoll edge */ c->write->handler = ngx_http_empty_handler;

        ngx_reusable_connection(c, 1);

        ngx_http_wait_request_handler(c->read);

        return;
    }
  ```
上面的过程核心就是SSL_get0_alpn_selected，这个函数在ssl 协议中取出alpn，如果是h2就设置connection 回调为h2,否则设置为ngx_http_wait_request_handler，会导致链接走到h1.

### 3. 客户端如何设置的alpn (应用层协议协商)
alpn 全程是 application_layer_protocol_negotiation, 如果不带，对于多端口支持的server，会走http1.x，这也能理解。

客户端如何设置的这个字段呢？客户端是golang 框架，如果用grpc.NewTls 默认设置, 核心实现就是AppendH2ToNextProtos。
```
// NewTLS uses c to construct a TransportCredentials based on TLS.
func NewTLS(c *tls.Config) TransportCredentials {
	tc := &tlsCreds{credinternal.CloneTLSConfig(c)}
	tc.config.NextProtos = credinternal.AppendH2ToNextProtos(tc.config.NextProtos)
	// If the user did not configure a MinVersion and did not configure a
	// MaxVersion < 1.2, use MinVersion=1.2, which is required by
	// https://datatracker.ietf.org/doc/html/rfc7540#section-9.2
	if tc.config.MinVersion == 0 && (tc.config.MaxVersion == 0 || tc.config.MaxVersion >= tls.VersionTLS12) {
		tc.config.MinVersion = tls.VersionTLS12
	}
	// If the user did not configure CipherSuites, use all "secure" cipher
	// suites reported by the TLS package, but remove some explicitly forbidden
	// by https://datatracker.ietf.org/doc/html/rfc7540#appendix-A
	if tc.config.CipherSuites == nil {
		for _, cs := range tls.CipherSuites() {
			if _, ok := tls12ForbiddenCipherSuites[cs.ID]; !ok {
				tc.config.CipherSuites = append(tc.config.CipherSuites, cs.ID)
			}
		}
	}
	return tc
}
```

为什么客户出错，就是没用grpc 框架默认的NewTls 方法，有字段缺失，导致握手的时候没带application_layer_protocol_negotiation 扩展字段。
