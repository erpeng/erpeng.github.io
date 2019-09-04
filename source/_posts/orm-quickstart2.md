---
title: ORM快速入门2
date: 2019-09-04
tags: PHP
---
>本文通过Laravel的illuminate来说明如何实现一个ORM


## 概述
一个ORM的实现一部分是连接,一部分是sql语句的生成,我们逐一讲解

## 连接

illuminate中连接选取涉及到了如下class:

![connection](/img/orm1.png)

php框架初始化时会首先实例化一个Illuminate\Database\Capsule\Manager,该class中包括一个\Illuminate\Database\DatabaseManager,DatabaseManager实际管理数据库的连接.其中有一个ConnectionFactory的实例,会根据配置调用不同的driver生成不同类型的连接.

php框架初始化完成后会调用\Capsule\Manager的bootEloquent()方法,该方法会设置属性resolver为DatabaseManager.

```
    public function bootEloquent()
    {
        Eloquent::setConnectionResolver($this->manager);
        ...
    }

    public static function setConnectionResolver(Resolver $resolver)
    {
        static::$resolver = $resolver;
    }

```

继续看Eloquent\Model中的getConnection()方法如何获取一个连接:

```
    public function getConnection()
    {
        return static::resolveConnection($this->getConnectionName());
    }

    public function getConnectionName()
    {
        return $this->connection;
    }
    
    public static function resolveConnection($connection = null)
    {
        return static::$resolver->connection($connection);
    }

```
可以看到,最终根据A(参看上篇文章示例class)中的connection字段获取到连接的name之后,调用DatabaseManager的connectionFactory工厂实例生成不同类型的数据库连接.

## sql生成

sql生成涉及到的类:

![query](/img/orm2.png)

通过上图以及orm系列第一篇文章我们知道whereInt最终调用的是Illuminate\Database\Query\Builder中的方法,如下:

```
    public function whereIn($column, $values, $boolean = 'and', $not = false)
    {
        $type = $not ? 'NotIn' : 'In';
        ...
        if ($values instanceof Arrayable) {
            $values = $values->toArray();
        }

        $this->wheres[] = compact('type', 'column', 'values', 'boolean');

        $this->addBinding($values, 'where');

        return $this;
    }
```
该函数只是对一些属性做赋值操作,包括wheres以及bindings,接着看get()函数:
```
public function get($columns = ['*'])
    {
        $original = $this->columns;

        if (is_null($original)) {
            $this->columns = $columns;
        }

        $results = $this->processor->processSelect($this, $this->runSelect());

        $this->columns = $original;

        return $results;
    }

   
    protected function runSelect()
    {
        return $this->connection->select($this->toSql(), $this->getBindings(), ! $this->useWritePdo);
    }

    public function toSql()
    {
        return $this->grammar->compileSelect($this);
    }
```
关键部分是$this->connection->select()以及$this->toSql().后者调用grammar的compileSelect生成sql.
具体调用链为compileSelect->compileComponents->compileWheres->whereIn

```
    protected function whereIn(Builder $query, $where)
    {
        if (empty($where['values'])) {
            return '0 = 1';
        }

        $values = $this->parameterize($where['values']);

        return $this->wrap($where['column']).' in ('.$values.')';
    }
```
大体来说就是一些格式化和拼接操作.

## 小结

至此,该ORM系列分析完毕.
