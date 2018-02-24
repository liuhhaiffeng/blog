---
title: error context call back
comments: true
date: 2018-02-24 22:39:56
tags: postgres
categories: [技术分享]
---

> In postgres kernel developing, error context call back is a common way to report an error message with additional context such as which the upper level function reach here. But when not handled carefully will lead to a serious bug.

The problem
-------------

the following code might lead to a core dump later on which is very hard to debug.

```c
struct my_func_ctx_arg
{
    bool do_it;
};
 
static void
my_func_ctx_callback(void *arg)
{
    struct my_func_ctx_arg *ctx_arg = arg;
    errcontext("during my_func(do_it=%d)", ctx_arg->do_it);
}

void
my_func(bool do_it)
{
    ErrorContextCallback myerrcontext;
    struct my_func_ctx_arg ctxinfo;

    ctxinfo.do_it = do_it;
    myerrcontext.callback = my_func_ctx_callback;
    myerrcontext.arg = &ctxinfo;
    myerrcontext.previous = error_context_stack;
    error_context_stack = &myerrcontext;

    if (!do_it)
    {
        elog(WARNING, "not doing it!");
        return;
    }

    do_the_thing();

    Assert(error_context_stack == &myerrcontext);
    error_context_stack = myerrcontext.previous;
}

```
Because myerrcontext is a local variable allocated on stack, upon my_func return, it will be freed. When this happens, error_context_stack will point to an invalid address. Later, when we want to recover the error_context_stack it got a corrupted pointer out of myerrcontext.previous which lead to the final crash.

The prevention
---------------

add extra checking code in elog.c
```c
void
verify_errcontext_stack(void)
{
    ErrorContextCallback *econtext;
    for (econtext = error_context_stack;
         econtext != NULL;
         econtext = econtext->previous)
    {
        Assert(econtext != NULL);
#ifdef USE_VALGRIND
        VALGRIND_CHECK_VALUE_IS_DEFINED(econtext);
        Assert(VALGRIND_CHECK_MEM_IS_ADDRESSABLE(econtext, sizeof(ErrorContextCallback)) == 0);
        VALGRIND_CHECK_VALUE_IS_DEFINED(econtext->previous);
        if (econtext->previous != NULL)
        {
            VALGRIND_CHECK_MEM_IS_ADDRESSABLE(econtext->previous, sizeof(ErrorContextCallback));
            VALGRIND_CHECK_VALUE_IS_DEFINED(econtext->previous->callback);
            VALGRIND_CHECK_VALUE_IS_DEFINED(econtext->previous->previous);
        }
#endif
        Assert(econtext->previous == NULL
               || econtext->previous->callback != NULL);
    }
}
```
The above code will help you to detect the bug before its too late to find out.

The fix
----------
1. alloc myerrcontext on heap using palloc
```c

    /* may be you have to switch to the error memory context */
    ErrorContextCallback *myerrcontext = palloc(sizeof(*myerrcontext));
    struct my_func_ctx_arg ctxinfo;

    ctxinfo.do_it = do_it;
    myerrcontext->callback = my_func_ctx_callback;
    myerrcontext->arg = &ctxinfo;
    myerrcontext->previous = error_context_stack;
    error_context_stack = myerrcontext;
```
2. do not return
```c
    if (do_it)
        do_the_thing();
    else
        elog(WARNING, "not doing it!");
```

Reference
-----------

https://blog.2ndquadrant.com/dev-corner-error-context-stack-corruption/

如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
