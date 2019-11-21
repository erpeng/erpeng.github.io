---
title: c和lua
date: 2019-11-19 
tags:
 - c
 - lua 
---
## An Overview of the C API

* C API既可以将lua作为library供c代码调用,也可以将c代码作为libraray供lua调用
* c 和 lua之间通过一个virtual stack交流.所有的数据交换和API调用都发生在stack,stack上也可以保存一些中间变量.该stack主要解决c和lua之间的两个问题:
    * lua是garbage collected,c需要显示的deallocation
    * lua是dynamic typing而c是static typing

## lua interpreter

```
    #include <stdio.h>
    #include <string.h>
    #include <lua.h>
    #include <lauxlib.h>
    #include <lualib.h>
    
    int main (void) {
      char buff[256];
      int error;
      lua_State *L = lua_open();   /* opens Lua */
      luaopen_base(L);             /* opens the basic library */
      luaopen_table(L);            /* opens the table library */
      luaopen_io(L);               /* opens the I/O library */
      luaopen_string(L);           /* opens the string lib. */
      luaopen_math(L);             /* opens the math lib. */
    
      while (fgets(buff, sizeof(buff), stdin) != NULL) {
        error = luaL_loadbuffer(L, buff, strlen(buff), "line") ||
                lua_pcall(L, 0, 0, 0);
        if (error) {
          fprintf(stderr, "%s", lua_tostring(L, -1));
          lua_pop(L, 1);  /* pop error message from the stack */
        }
      }
    
      lua_close(L);
      return 0;
    }
```
* lua.h 定义lua的basic functions,所有函数以lua_开头
* lauxlib.h定义auxlib.以luaL_开头,使用basic function定义一些更高层级抽象的函数
* lua_open打开一个fresh environment,没有任何预定义的函数.所有的标准库通过lualib.h定义的函数去加载
* luaL_loadbuffer编译lua代码,没有任何错误的话返回0并且将其放入stack,lua_pcall弹出该代码然后运行.如果有错误,会将错误押入stack,lua_tostring和lua_pop操作并弹出错误

## The Stack

lua代码 
```
a[k] = v
```
a,k,v的类型都不确定.如果在C中按所有类型都写一遍settable会造成组合爆炸,当然,可以定义一个union例如lua_Value,写为如下:
```
void lua_settable (lua_Value a, lua_Value k, lua_Value v);
```
**一来太复杂,不便与lua和其他语言交互,二来垃圾回收也不好处理**
因此使用一个stack.**stack相当于将c和lua解耦合,只需要写不同类型的c函数处理即可.而且stack由lua管理,便于垃圾回收**

注意lua操作时严格按LIFO,C code无此限制,可以自由操作

## pushing elements

```
    void lua_pushnil (lua_State *L);
    void lua_pushboolean (lua_State *L, int bool);
    void lua_pushnumber (lua_State *L, double n);
    void lua_pushlstring (lua_State *L, const char *s,
                                        size_t length);
    void lua_pushstring (lua_State *L, const char *s);

```
每种lua类型都有对应的c函数压入元素.lua每次都会对数据进行拷贝而不会使用指针(除了C functions).因此函数返回后可以free或者modify该buffer,不影响lua环境中的数据
每次lua调用c时,注意stack有20 free slots,因此需要自己用如下函数检查是否够用 
```
int lua_checkstack(lua_State *L,int sz)
```

## Querying Elements

stack从top到bottom的索引依次为-1,-2,-3...
如果从bottom到top,按1,2,3索引

```
    int            lua_toboolean (lua_State *L, int index);
    double         lua_tonumber (lua_State *L, int index);
    const char    *lua_tostring (lua_State *L, int index);
    size_t         lua_strlen (lua_State *L, int index);
```

```
    int lua_is... (lua_State *L, int index);
    lua_type
```

```
    int   lua_gettop (lua_State *L);
```
返回栈中元素个数

```
    void  lua_settop (lua_State *L, int index);
```
设置top元素为一个指定的值.lua_settop(L,0)清空stack.如果先前的top大于index,则清空top.反之push nil达到top

```
#define lua_pop(L,n) lua_settop(L,-(n)-1)
```

```
    void  lua_pushvalue (lua_State *L, int index);
    void  lua_remove (lua_State *L, int index);
    void  lua_insert (lua_State *L, int index);
    void  lua_replace (lua_State *L, int index);
```
lua_pushvalue拷贝index位置的元素到top,总长度+1
lua_remove将index位置的元素删除,总长度-1
lua_insert将top位置的元素放置到index位置,index位置以上的元素上移,总长度+1
lua_replace将top位置的元素弹出放置到index位置,总长度不变

## call function 

``` lua
    function f (x, y)
      return (x^2 * math.sin(y))/(1 - x)
    end

```
```c
    /* call a function `f' defined in Lua */
    double f (double x, double y) {
      double z;
    
      /* push functions and arguments */
      lua_getglobal(L, "f");  /* function to be called */
      lua_pushnumber(L, x);   /* push 1st argument */
      lua_pushnumber(L, y);   /* push 2nd argument */
    
      /* do the call (2 arguments, 1 result) */
      if (lua_pcall(L, 2, 1, 0) != 0)
        error(L, "error running function `f': %s",
                 lua_tostring(L, -1));
    
      /* retrieve result */
      if (!lua_isnumber(L, -1))
        error(L, "function `f' must return a number");
      z = lua_tonumber(L, -1);
      lua_pop(L, 1);  /* pop returned value */
      return z;
    }

```
lua_getglobal会将f放到stack中,继续将参数x,y分别压栈,然后调用lua_pcall,调用完毕,栈中top位置要不是error要不是返回结果.lua_pcall的参数分别代表L,参数个数,结果个数,错误处理函数索引(需要提前压入stack)


## calling table
```lua
    background = {r=0.30, g=0.10, b=0}

```

```c
    lua_getglobal(L, "background");
    if (!lua_istable(L, -1))
      error(L, "`background' is not a valid color table");
    
    red = getfield("r");
    green = getfield("g");
    blue = getfield("b");

        #define MAX_COLOR       255
    
    /* assume that table is on the stack top */
    int getfield (const char *key) {
      int result;
      lua_pushstring(L, key);
      lua_gettable(L, -2);  /* get background[key] */
      if (!lua_isnumber(L, -1))
        error(L, "invalid component in background color");
      result = (int)lua_tonumber(L, -1) * MAX_COLOR;
      lua_pop(L, 1);  /* remove number */
      return result;
    }

```
lua_gettable获取指定field的值,将field压栈,然后指定table的index,value会放到top位置

```c
    void setfield (const char *index, int value) {
      lua_pushstring(L, index);
      lua_pushnumber(L, (double)value/MAX_COLOR);
      lua_settable(L, -3);
    }
```
同理,lua_settable将key和value压栈后,指定table的index,即可设置一个table.结束后会将key/value弹出

```c
    void setcolor (struct ColorTable *ct) {
      lua_newtable(L);               /* creates a table */
      setfield("r", ct->red);        /* table.r = ct->r */
      setfield("g", ct->green);      /* table.g = ct->g */
      setfield("b", ct->blue);       /* table.b = ct->b */
      lua_setglobal(L, ct->name);    /* `name' = table */
    }

```
lua_newtable生成一个table并压栈,lua_setglobal弹出table并且将其赋值给指定名称



## 参考链接
* https://www.lua.org/pil/24.html