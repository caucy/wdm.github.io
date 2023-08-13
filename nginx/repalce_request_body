# 参考nginx-lua replace request body

## lua 有set_body_data 实现

lua 有set_body_data，可以实现reset body，一般在读完body 后使用。

使用方法如下：
```
ngx.req.set_body_data
syntax: ngx.req.set_body_data(data)

context: rewrite_by_lua*, access_by_lua*, content_by_lua*
```

## 有set_body_data 如何实现？

ngx_http_lua_inject_req_body_api 中注册了ngx_http_lua_ngx_req_set_body_data， 可以参考ngx_http_lua_ngx_req_set_body_data 的实现
修改r->request_body_bufs，除了修改request_body_bufs 还有什么需要注意的？

```
static int
ngx_http_lua_ngx_req_set_body_data(lua_State *L)
{
    ...

    // 第一步close request_body 中file_buf 和bufs 
    rb = r->request_body;
    tf = rb->temp_file;

    if (tf) {
        if (tf->file.fd != NGX_INVALID_FILE) {
            ngx_http_lua_pool_cleanup_file(r->pool, tf->file.fd);
            tf->file.fd = NGX_INVALID_FILE;
        }

        rb->temp_file = NULL;
    }

    if (body.len == 0) {
        if (rb->bufs) {

            for (cl = rb->bufs; cl; cl = cl->next) {
                if (cl->buf->tag == tag && cl->buf->temporary) {

                    dd("free old request body buffer: size:%d",
                       (int) ngx_buf_size(cl->buf));

                    ngx_pfree(r->pool, cl->buf->start);
                    cl->buf->tag = (ngx_buf_tag_t) NULL;
                    cl->buf->temporary = 0;
                }
            }
        }

        rb->bufs = NULL;
        rb->buf = NULL;

        dd("request body is set to empty string");
        goto set_header;
    }

// 第二步，换掉 request_body->bufs 成新的buf，并将temporary 置1
    if (rb->bufs) {

        for (cl = rb->bufs; cl; cl = cl->next) {
            if (cl->buf->tag == tag && cl->buf->temporary) {
                dd("free old request body buffer: size:%d",
                   (int) ngx_buf_size(cl->buf));

                ngx_pfree(r->pool, cl->buf->start);
                cl->buf->tag = (ngx_buf_tag_t) NULL;
                cl->buf->temporary = 0;
            }
        }

        rb->bufs->next = NULL;

        b = rb->bufs->buf;

        ngx_memzero(b, sizeof(ngx_buf_t));

        b->temporary = 1;
        b->tag = tag;

        b->start = ngx_palloc(r->pool, body.len);
        if (b->start == NULL) {
            return luaL_error(L, "no memory");
        }

        b->end = b->start + body.len;

        b->pos = b->start;
        b->last = ngx_copy(b->pos, body.data, body.len);

    } else {

        rb->bufs = ngx_alloc_chain_link(r->pool);
        if (rb->bufs == NULL) {
            return luaL_error(L, "no memory");
        }

        rb->bufs->next = NULL;

        b = ngx_create_temp_buf(r->pool, body.len);
        if (b == NULL) {
            return luaL_error(L, "no memory");
        }

        b->tag = tag;
        b->last = ngx_copy(b->pos, body.data, body.len);

        rb->bufs->buf = b;
        rb->buf = b;
    }

// 改header 头，重置content-length
set_header:

    /* override input header Content-Length (value must be null terminated) */

    value.data = ngx_palloc(r->pool, NGX_SIZE_T_LEN + 1);
    if (value.data == NULL) {
        return luaL_error(L, "no memory");
    }

    value.len = ngx_sprintf(value.data, "%uz", body.len) - value.data;
    value.data[value.len] = '\0';

    dd("setting request Content-Length to %.*s (%d)",
       (int) value.len, value.data, (int) body.len);

    r->headers_in.content_length_n = body.len;

    if (r->headers_in.content_length) {
        r->headers_in.content_length->value.data = value.data;
        r->headers_in.content_length->value.len = value.len;

    } else {

        ngx_str_set(&key, "Content-Length");

        rc = ngx_http_lua_set_input_header(r, key, value, 1 /* override */);
        if (rc != NGX_OK) {
            return luaL_error(L, "failed to reset the Content-Length "
                              "input header");
        }
    }

    return 0;
}
```

## replace 关键步骤

1. close request_body 中file_buf 和bufs 

2. 换掉 request_body->bufs 成新的buf，并将temporary 置1, 表示数据在内存里

3. 修改header_in，重置content-length。ngx_http_lua_set_input_header 可以省掉查写hash表的复杂过程

