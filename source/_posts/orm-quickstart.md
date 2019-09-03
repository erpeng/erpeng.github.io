---
title: ORM快速入门
date: 2019-09-03
tags: PHP
---
>本文通过Laravel的illuminate来说明如何实现一个ORM

## ORM写法


首先定义class A如下:
```
use Illuminate\Database\Eloquent\Model as Model;

class A extends Model
{
    /**
     * The table associated with the model.
     *
     * @var string
     */
    protected $table = 'table_a';
}

```
其中class A中定义了一个table属性,值为'table_a'

然后php中通过如下方法获取table_a中的数据

```
$rows = A::query()->whereIn('id', [1,2,3])->get()

```
对应sql语句为
```
select * from table_a where id in (1,2,3);
```

## 实现思路

* 自底向上思考,数据库只认sql语句,所以最终需要将ORM代码映射为相应的sql并执行
* 执行时需要数据库连接,所以需要有一个connection对象(包括ip,port,username,password,db及其配置属性),并且connection对象实际也是可配置的,配置方法同table属性,和每一个model关联起来.如下:
```
    /**
     * The database connection associated with the model.
     *
     * @var string
     */
    protected $connection = 'history';
```
* 所以我们需要的只是连接和sql,其中连接可能是mysql,sqlite,sqlserver等,并且不同的数据库生成的sql也有一定的差异

## illuminate实现

从A::query()开始,调用链接如下:
```

    public static function query()
    {
        return (new static)->newQuery();
    }


    public function newQuery()
    {
        $builder = $this->newQueryWithoutScopes();
        ...
        return $builder;
    }
```
query()返回值为\Illuminate\Database\Eloquent\Builder，调用时使用new static,为延迟静态绑定,所以下文中的$this都为class A的实例对象


生成Eloquent\Builder的函数如下,\Illuminate\Database\Eloquent\Builder中有一个query属性为\Illuminate\Database\Query\Builder,Query\Builder下文再述
```
    public function newQueryWithoutScopes()
    {
        $builder = $this->newEloquentBuilder(
            $this->newBaseQueryBuilder()
        );

        return $builder->setModel($this)->with($this->with);
    }
    protected function newBaseQueryBuilder()
    {
        $conn = $this->getConnection();

        $grammar = $conn->getQueryGrammar();

        return new QueryBuilder($conn, $grammar, $conn->getPostProcessor());
    }
```

注意newQueryWithoutScopes函数中最后的setModel函数,传入的参数为$this,即A的一个实例;下文代码中getTable获取表名时如果A中设置了table属性,则直接实用其属性值,即上述配置中的table_a,否则将类名格式化之后作为表名.
```
    public function setModel(Model $model)
    {
        $this->model = $model;

        $this->query->from($model->getTable());

        return $this;
    }


    public function getTable()
    {
        if (isset($this->table)) {
            return $this->table;
        }

        return str_replace('\\', '', Str::snake(Str::plural(class_basename($this))));
    }
```

继续看whereIn函数,\Illuminate\Database\Eloquent\Builder中有一个魔术方法__call,whereIn实际会调用Eloquent\Builder中query属性,即\Illuminate\Database\Query\Builder的whereIn方法 

get()函数同理也是调用query的get()方法,最终首先生成sql,然后通过mysql连接执行并返回结果

## 小结
后文详述生成sql以及连接生成相关的代码

