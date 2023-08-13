# 参考body_filter_by_lua_block 实现替换response body

## body_filter_by_lua_block 的使用
```
syntax: body_filter_by_lua_block { lua-script-str }

context: http, server, location, location if

phase: output-body-filter
```

在body_filter_by_lua_block阶段执行<lua-script-str>的代码，用于设置输出响应体的过滤器。在此阶段可以修改响应体的内容，
如修改字母的大小写、替换字符串等。

通过ngx.arg[1] 输入数据流，结束标识eof是响应体数据的最后一位ngx.arg[2]（ngx.arg[2]是Lua的布尔值类型的数据）。
repsonse 如果没开启buffer，此阶段的Lua指令也会被执行多次。

## bodyfilter 如何实现替换response

ngx_http_lua_body_filter 用新buf 替代body_filter 的input chain
```
static ngx_int_t
ngx_http_lua_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    if (llcf->body_filter_handler == NULL || r->header_only) {
        dd("no body filter handler found");
        return ngx_http_next_body_filter(r, in);
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    dd("ctx = %p", ctx);

    if (ctx == NULL) {
        ctx = ngx_http_lua_create_ctx(r);
        if (ctx == NULL) {
            return NGX_ERROR;
        }
    }

    if (ctx->seen_last_in_filter) {
        for (/* void */; in; in = in->next) {
            dd("mark the buf as consumed: %d", (int) ngx_buf_size(in->buf));
            in->buf->pos = in->buf->last;
            in->buf->file_pos = in->buf->file_last;
        }

        in = NULL;

        /* continue to call ngx_http_next_body_filter to process cached data */
    }

    if (in != NULL
        && ngx_chain_add_copy(r->pool, &ctx->filter_in_bufs, in) != NGX_OK)
    {
        return NGX_ERROR;
    }

    if (ctx->filter_busy_bufs != NULL
        && (r->connection->buffered
            & (NGX_HTTP_LOWLEVEL_BUFFERED | NGX_LOWLEVEL_BUFFERED)))
    {
        /* Socket write buffer was full on last write.
         * Try to write the remain data, if still can not write
         * do not execute body_filter_by_lua otherwise the `in` chain will be
         * replaced by new content from lua and buf of `in` mark as consumed.
         * And then ngx_output_chain will call the filter chain again which
         * make all the data cached in the memory and long ngx_chain_t link
         * cause CPU 100%.
         */
        rc = ngx_http_next_body_filter(r, NULL);

        if (rc == NGX_ERROR) {
            return rc;
        }

        out = NULL;
        ngx_chain_update_chains(r->pool,
                                &ctx->free_bufs, &ctx->filter_busy_bufs, &out,
                                (ngx_buf_tag_t) &ngx_http_lua_body_filter);
        if (rc != NGX_OK
            && ctx->filter_busy_bufs != NULL
            && (r->connection->buffered
                & (NGX_HTTP_LOWLEVEL_BUFFERED | NGX_LOWLEVEL_BUFFERED)))
        {
            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "waiting body filter busy buffer to be sent");
            return NGX_AGAIN;
        }

        /* continue to process bufs in ctx->filter_in_bufs */
    }

    if (ctx->cleanup == NULL) {
        cln = ngx_pool_cleanup_add(r->pool, 0);
        if (cln == NULL) {
            return NGX_ERROR;
        }

        cln->handler = ngx_http_lua_request_cleanup_handler;
        cln->data = ctx;
        ctx->cleanup = &cln->handler;
    }

    old_context = ctx->context;
    ctx->context = NGX_HTTP_LUA_CONTEXT_BODY_FILTER;

    in = ctx->filter_in_bufs;
    ctx->filter_in_bufs = NULL;

    if (in != NULL) {
        dd("calling body filter handler");
        rc = llcf->body_filter_handler(r, in);

        dd("calling body filter handler returned %d", (int) rc);

        ctx->context = old_context;

        if (rc != NGX_OK) {
            return NGX_ERROR;
        }

        lmcf = ngx_http_get_module_main_conf(r, ngx_http_lua_module);

        /* lmcf->body_filter_chain is the new buffer chain if
         * body_filter_by_lua set new body content via ngx.arg[1] = new_content
         * otherwise it is the original `in` buffer chain.
         */
        out = lmcf->body_filter_chain;

        if (in != out) {
            /* content of body was replaced in
             * ngx_http_lua_body_filter_param_set and the buffers was marked
             * as consumed.
             */
            for (cl = in; cl != NULL; cl = ln) {
                ln = cl->next;
                ngx_free_chain(r->pool, cl);
            }

            if (out == NULL) {
                /* do not forward NULL to the next filters because the input is
                 * not NULL */
                return NGX_OK;
            }
        }

    } else {
        out = NULL;
    }

    rc = ngx_http_next_body_filter(r, out);
    if (rc == NGX_ERROR) {
        return NGX_ERROR;
    }

    ngx_chain_update_chains(r->pool,
                            &ctx->free_bufs, &ctx->filter_busy_bufs, &out,
                            (ngx_buf_tag_t) &ngx_http_lua_body_filter);

    return rc;
}
```


## 替换body 需要注意的事项
1. body_filter 是要可重入的
2. 失败要用ngx_http_next_body_filter
3. 用新chain ngx_http_next_body_filter 可以替换掉老的body
4. 如果大小变化了，要补充实现header_filter

