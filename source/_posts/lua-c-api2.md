---
title: c和lua-2
date: 2019-11-24 
tags:
 - c
 - lua 
---
## C functions

c函数必须按一定的签名来写,如下:
```c
    typedef int (*lua_CFunction) (lua_State *L);
```
例如从lua中调用c写的sin函数,如下:
```c
    static int l_sin (lua_State *L) {
      double d = lua_tonumber(L, 1);  /* get argument */
      lua_pushnumber(L, sin(d));  /* push result */
      return 1;  /* number of results */
    }

```
至于如何将函数注册入lua,下个章节介绍
接着是一个遍历目录的例子,如下:
```c
 #include <dirent.h>
    #include <errno.h>
    
    static int l_dir (lua_State *L) {
      DIR *dir;
      struct dirent *entry;
      int i;
      const char *path = luaL_checkstring(L, 1); //检查输入参数
    
      /* open directory */
      dir = opendir(path);
      if (dir == NULL) {  /* error opening the directory? */
        lua_pushnil(L);  /* return nil and ... */ //失败情形会返回一个nil和一个错误信息
        lua_pushstring(L, strerror(errno));  /* error message */
        return 2;  /* number of results */
      }
    
      /* create result table */
      lua_newtable(L);
      i = 1;
      while ((entry = readdir(dir)) != NULL) {
        lua_pushnumber(L, i++);  /* push key */
        lua_pushstring(L, entry->d_name);  /* push value */
        lua_settable(L, -3);//注意每次循环中settable之后会自动将index和value弹出.所以table还是在栈顶
      }
    
      closedir(dir);
      return 1;  /* table is already on top */
    }
```
注意lua中的错误处理,如果lua_newtable,lua_pushstring和lua_settable可能会内存不足导致报错然后直接interrupt该函数的执行,此时不会执行到closedir造成内存泄漏,后续章节会介绍一个更好的方法

## C libraries
注册上边的l_dir函数进入lua.
动态加载,如下:
```c
    //定义一个luaL_reg类型的数组,数组第一个元素为在lua中的名称,第二个元素为函数指针
    static const struct luaL_reg mylib [] = {
      {"dir", l_dir},
      {NULL, NULL}  /* sentinel */
    };
    
    //定义一个函数,函数签名必须是lua定义好的签名格式,调用luaL_openlib,该函数第二个参数为需要注册的函数在lua中的library name,第三个参数为上边的luaL_reg数组,第四个参数为upvalues,本例中为0
    int luaopen_mylib (lua_State *L) {
      luaL_openlib(L, "mylib", mylib, 0);
      return 1;//return 1说明返回一个结果,为该library
    }
    //加载动态库.会将luaopen_mysqlib函数注册入lua,名称为mylib,然后执行mylib,会调用luaopen_mylib.有点绕..
    mylib = loadlib("fullname-of-your-library", "luaopen_mylib")
```
静态加载,首先定义如下头文件mylib.h,如下:
```c
    int luaopen_mylib (lua_State *L);
    #define LUA_EXTRALIBS { "mylib", luaopen_mylib },
```
重新编译lua,加载入该头文件mylib.h


## 技巧

### table manipulation
```c
    void lua_rawgeti (lua_State *L, int index, int key);
    void lua_rawseti (lua_State *L, int index, int key);

```
index是table的位置,key是key的位置,lua_rawgeti相当于:
```
    lua_pushnumber(L, key);
    lua_rawget(L, index);
```
lua_rawseti相当于:
```c
    lua_pushnumber(L, key);
    lua_insert(L, -2);  /* put `key' below previous value */
    lua_rawset(L, t);
```

```c
  int l_map (lua_State *L) {
      int i, n;
    
      /* 1st argument must be a table (t) */
      luaL_checktype(L, 1, LUA_TTABLE);
    
      /* 2nd argument must be a function (f) */
      luaL_checktype(L, 2, LUA_TFUNCTION);
    
      n = luaL_getn(L, 1);  /* get size of table */
    
      for (i=1; i<=n; i++) {
        lua_pushvalue(L, 2);   /* push f */
        lua_rawgeti(L, 1, i);  /* push t[i] */
        lua_call(L, 1, 1);     /* call f(t[i]) */
        lua_rawseti(L, 1, i);  /* t[i] = result */
      }
    
      return 0;  /* no results */
    }
```
l_map函数,最后的lua_rawseti函数原理为将1处(table)key为i的值设置为top处的value


### string manipulation

```c
    lua_pushlstring(L, s+i, j-i+1);
```
```c
  lua_concat(L, n)
```
将stack顶部的n个string concat,然后放到top位置

```c
    const char *lua_pushfstring (lua_State *L,
                                 const char *fmt, ...);
```
类似sprintf,会将格式化的string放到top位置

buffer操作:
```c
    //buffer初始化
    luaL_Buffer b;
    luaL_buffinit(L, &b);

    //具体操作函数
    void luaL_buffinit (lua_State *L, luaL_Buffer *B);
    void luaL_putchar (luaL_Buffer *B, char c);
    void luaL_addlstring (luaL_Buffer *B, const char *s,
                                          size_t l);
    void luaL_addstring (luaL_Buffer *B, const char *s);
    void luaL_pushresult (luaL_Buffer *B);
```

示例:
```c
    static int str_upper (lua_State *L) {
      size_t l;
      size_t i;
      luaL_Buffer b;
      const char *s = luaL_checklstr(L, 1, &l);
      luaL_buffinit(L, &b);
      for (i=0; i<l; i++)
        luaL_putchar(&b, toupper((unsigned char)(s[i])));
      luaL_pushresult(&b);
      return 1;
    }
```

## 参考链接
* https://www.lua.org/pil/24.html