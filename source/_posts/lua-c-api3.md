---
title: c和lua-3
date: 2019-11-24 
tags:
 - c
 - lua 
---
## c函数的状态
一般在c函数中保存non-local data时可以使用全局或者静态变量.但如果c library供lua调用的话,这种方法是不可行的(一则涉及lua和c的类型转换,二者此类函数不能被多个lua state使用).此时可以将其保存为lua global variable,但这样lua代码也是可以操作这些数据的.因此lua提供了一个独立的table,叫做registry,c能自由使用但lua不能.

### The registry

```c
    lua_pushstring(L, "Key");
    lua_gettable(L, LUA_REGISTRYINDEX);
```
伪索引,LUA_REGISTRYINDEX指向了registry,如上代码取出field为Key的value值

### References

reference system由auxlib中的一批函数组成,其作用在于使用者不需要担心生成的key是否唯一(防止在registry中同样的key被覆盖),例如如下函数:
```c
    int r = luaL_ref(L, LUA_REGISTRYINDEX);

```
会将栈顶的值放入registry,key为r,r是一个整形数字.所以如果不使用reference system,不要使用整形数作为registry的key.
r被称为一个reference.
反向操作会取出registry中key为r的值,并且放到栈中
```c   
  lua_rawgeti(L, LUA_REGISTRYINDEX, r);
```
如下操作释放key和value
```c
    luaL_unref(L, LUA_REGISTRYINDEX, r);
```

### Upvalues

registry解决global value的问题,upvalues解决static value的问题,所以只对特定的函数可见.c函数和upvalues组合为一个closure.
例如:
```c
    /* forward declaration */
    static int counter (lua_State *L);
    
    int newCounter (lua_State *L) {
      lua_pushnumber(L, 0);
      lua_pushcclosure(L, &counter, 1);
      return 1;
    }
```
lua_pushcclosure将一个c函数指针以及一个upvalues压入,首先将一个upvalue的初始值(本例中为0)压入.

counter的定义如下:
```c
    static int counter (lua_State *L) {
      double val = lua_tonumber(L, lua_upvalueindex(1));
      lua_pushnumber(L, ++val);  /* new value */
      lua_pushvalue(L, -1);  /* duplicate it */
      lua_replace(L, lua_upvalueindex(1));  /* update upvalue */
      return 1;  /* return new value */
    }
```
lua_upvalueindex会将upvalue取出,取出之后+1然后压栈,复制一份该值然后用该值更新upvalue,最后将+1的值返回.

## User-Defined Types In C
前边我们讲了如何使用c函数扩展lua,本节讲解如何使用c结构扩展lua
例如如下一个double类型的数组结构:
```c
    typedef struct NumArray {
      int size;
      double values[1];  /* variable part */
    } NumArray;
```

### Userdata

如下函数申请size大小的内存,然后将指针放到栈顶,lua中的userdata类型
```c
    void *lua_newuserdata (lua_State *L, size_t size);
```

申请空间:

```c
    static int newarray (lua_State *L) {
      int n = luaL_checkint(L, 1);
      size_t nbytes = sizeof(NumArray) + (n - 1)*sizeof(double);
      NumArray *a = (NumArray *)lua_newuserdata(L, nbytes);
      a->size = n;
      return 1;  /* new userdatum is already on the stack */
    }
```

设置数据:
```c
    static int setarray (lua_State *L) {
      NumArray *a = (NumArray *)lua_touserdata(L, 1);
      int index = luaL_checkint(L, 2);
      double value = luaL_checknumber(L, 3);
    
      luaL_argcheck(L, a != NULL, 1, "`array' expected");
    
      luaL_argcheck(L, 1 <= index && index <= a->size, 2,
                       "index out of range");
    
      a->values[index-1] = value;
      return 0;
    }
```
获取数据:
```c
    static int getarray (lua_State *L) {
      NumArray *a = (NumArray *)lua_touserdata(L, 1);
      int index = luaL_checkint(L, 2);
    
      luaL_argcheck(L, a != NULL, 1, "`array' expected");
    
      luaL_argcheck(L, 1 <= index && index <= a->size, 2,
                       "index out of range");
    
      lua_pushnumber(L, a->values[index-1]);
      return 1;
    }
```
获取数组大小:
```c
    static int getsize (lua_State *L) {
      NumArray *a = (NumArray *)lua_touserdata(L, 1);
      luaL_argcheck(L, a != NULL, 1, "`array' expected");
      lua_pushnumber(L, a->size);
      return 1;
    }
```

注册进lua:
```c
    static const struct luaL_reg arraylib [] = {
      {"new", newarray},
      {"set", setarray},
      {"get", getarray},
      {"size", getsize},
      {NULL, NULL}
    };
    
    int luaopen_array (lua_State *L) {
      luaL_openlib(L, "array", arraylib, 0);
      return 1;
    }
```
使用:
```
    a = array.new(1000)
    print(a)               --> userdata: 0x8064d48
    print(array.size(a))   --> 1000
    for i=1,1000 do
      array.set(a, i, 1/i)
    end
    print(array.get(a, 10))  --> 0.1

```

### metatables

```c
    int   luaL_newmetatable (lua_State *L, const char *tname);
    void  luaL_getmetatable (lua_State *L, const char *tname);
    void *luaL_checkudata (lua_State *L, int index,
                                         const char *tname);
```
上述三个函数分别生成一个metatable(名称为tname),获取该metable,并且检测index位置处的userdata是否匹配tname指定的metatable,不匹配返回null,否则返回userdata的地址
lua_setmetatable将指定index的元素的metatable设置为栈顶的metatable

### Object-Oriented Access
为了使用面向对象的语法,例如:
```lua
    a = array.new(1000)
    print(a:size())     --> 1000
    a:set(10, 3.4)
    print(a:get(10))    --> 3.4
```
我们需要将__index设置一下:
```lua
    local metaarray = getmetatable(array.new(1))
    metaarray.__index = metaarray
    metaarray.set = array.set
    metaarray.get = array.get
    metaarray.size = array.size
```
那么在c中可以按如下方法:
```c
    static const struct luaL_reg arraylib_f [] = {
      {"new", newarray},
      {NULL, NULL}
    };
    
    static const struct luaL_reg arraylib_m [] = {
      {"set", setarray},
      {"get", getarray},
      {"size", getsize},
      {NULL, NULL}
    };
    int luaopen_array (lua_State *L) {
      luaL_newmetatable(L, "LuaBook.array");
    
      lua_pushstring(L, "__index");
      lua_pushvalue(L, -2);  /* pushes the metatable */
      lua_settable(L, -3);  /* metatable.__index = metatable */
    
      luaL_openlib(L, NULL, arraylib_m, 0);
    
      luaL_openlib(L, "array", arraylib_f, 0);
      return 1;
    }
```

### Array Access

为了使用数组语法,例如a:get(i)为a[i],可以如下设置
```lua
    local metaarray = getmetatable(newarray(1))
    metaarray.__index = array.get
    metaarray.__newindex = array.set
```
c中为:
```c
int luaopen_array (lua_State *L) {
      luaL_newmetatable(L, "LuaBook.array");
      luaL_openlib(L, "array", arraylib, 0);
    
      /* now the stack has the metatable at index 1 and
         `array' at index 2 */
      lua_pushstring(L, "__index");
      lua_pushstring(L, "get");
      lua_gettable(L, 2);  /* get array.get */
      lua_settable(L, 1);  /* metatable.__index = array.get */
    
      lua_pushstring(L, "__newindex");
      lua_pushstring(L, "set");
      lua_gettable(L, 2); /* get array.set */
      lua_settable(L, 1); /* metatable.__newindex = array.set */
    
      return 0;
    }
```

### Light Userdata

light userdata是一个函数指针

```c
    void lua_pushlightuserdata (lua_State *L, void *p);
```

##  Managing Resources
full userdata的资源回收由lua管理,只是一段内存.light userdata不由lua管理,还有一些其他资源例如file descriptors,由lua的__gc metatable管理,类似于finalizer或者destructor

修改目录遍历函数,如下:
```c
int luaopen_dir (lua_State *L) {
      luaL_newmetatable(L, "LuaBook.dir");
    
      /* set its __gc field */
      lua_pushstring(L, "__gc");
      lua_pushcfunction(L, dir_gc);
      lua_settable(L, -3);
    
      /* register the `dir' function */
      lua_pushcfunction(L, l_dir);
      lua_setglobal(L, "dir");
    
      return 0;
    }
```
设置LuaBook.dir的__gc字段为dir_gc,dir_gc负责回收Dir这个资源.设置一个全局变量dir为l_dir函数.

```c
    static int dir_gc (lua_State *L) {
      DIR *d = *(DIR **)lua_touserdata(L, 1);
      if (d) closedir(d);
      return 0;
    }
```
Dir资源为一个userdata
```c
    #include <dirent.h>
    #include <errno.h>
    
    /* forward declaration for the iterator function */
    static int dir_iter (lua_State *L);
    
    static int l_dir (lua_State *L) {
      const char *path = luaL_checkstring(L, 1);
    
      /* create a userdatum to store a DIR address */
      DIR **d = (DIR **)lua_newuserdata(L, sizeof(DIR *));
    
      /* set its metatable */
      luaL_getmetatable(L, "LuaBook.dir");
      lua_setmetatable(L, -2);
    
      /* try to open the given directory */
      *d = opendir(path);
      if (*d == NULL)  /* error opening the directory? */
        luaL_error(L, "cannot open %s: %s", path,
                                            strerror(errno));
    
      /* creates and returns the iterator function
         (its sole upvalue, the directory userdatum,
         is already on the stack top */
      lua_pushcclosure(L, dir_iter, 1);
      return 1;
    }
```
设置userdata Dir的metable为LuaBook.dir.然后打开目录,设置一个闭包,函数为dir_iter,upvalue为栈顶的userdata

```c
    static int dir_iter (lua_State *L) {
      DIR *d = *(DIR **)lua_touserdata(L, lua_upvalueindex(1));
      struct dirent *entry;
      if ((entry = readdir(d)) != NULL) {
        lua_pushstring(L, entry->d_name);
        return 1;
      }
      else return 0;  /* no more values to return */
    }
```
遍历目录

lua中使用方法如下:
```lua
    for fname in dir(".") do  print(fname)  end
```


## 参考链接
* https://www.lua.org/pil/24.html