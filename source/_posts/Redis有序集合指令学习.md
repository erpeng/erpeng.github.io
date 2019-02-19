---
title: Redis有序集合指令学习
date: 2018-07-11 13:21:01
tags:
---
## ZADD
ZADD key [NX|XX] [CH] [INCR]score member [score member ...]

将元素及对应分值添加到一个有序集合中

NX:不更新已经存在的key,只增加新元素

XX:只更新已经存在的key,不增加新元素

CH:abbr:changed.不指定时只返回新增的元素个数,指定时返回新增的和更新的元素个数之和

INCR:参考zincrby

```c
//通过第二个参数区分是zadd还是zincrby
void zaddCommand(client *c) {
    zaddGenericCommand(c,ZADD_NONE);
}
```

```c
/* This generic command implements both ZADD and ZINCRBY. */
//zadd和zincrby两个命令都是调用这个函数
void zaddGenericCommand(client *c, int flags) {
    static char *nanerr = "resulting score is not a number (NaN)";
    robj *key = c->argv[1];
    robj *zobj;
    sds ele;
    double score = 0, *scores = NULL;
    int j, elements;
    int scoreidx = 0;
    /* The following vars are used in order to track what the command actually
     * did during the execution, to reply to the client and to trigger the
     * notification of keyspace change. */
    int added = 0;      /* Number of new elements added. */
    int updated = 0;    /* Number of elements with updated score. */
    int processed = 0;  /* Number of elements processed, may remain zero with
                           options like XX. */

    /* Parse options. At the end 'scoreidx' is set to the argument position
     * of the score of the first score-element pair. */
    scoreidx = 2;//从第二个参数开始处理.先处理nx,xx,ch,incr参数
    while(scoreidx < c->argc) {
        char *opt = c->argv[scoreidx]->ptr;
        if (!strcasecmp(opt,"nx")) flags |= ZADD_NX;
        else if (!strcasecmp(opt,"xx")) flags |= ZADD_XX;
        else if (!strcasecmp(opt,"ch")) flags |= ZADD_CH;
        else if (!strcasecmp(opt,"incr")) flags |= ZADD_INCR;
        else break;
        scoreidx++;
    }

    /* Turn options into simple to check vars. */
    //从flag中取出相应的标志赋给独立的变量
    int incr = (flags & ZADD_INCR) != 0;
    int nx = (flags & ZADD_NX) != 0;
    int xx = (flags & ZADD_XX) != 0;
    int ch = (flags & ZADD_CH) != 0;

    /* After the options, we expect to have an even number of args, since
     * we expect any number of score-element pairs. */
    //member和score是一一对应的,所以肯定是2的倍数.所以如果不是2的倍数或者根本
    //没有member和score,直接返回命令语法错误
    elements = c->argc-scoreidx;
    if (elements % 2 || !elements) {
        addReply(c,shared.syntaxerr);
        return;
    }
    //elements赋值为有多少对<element,score>
    elements /= 2; /* Now this holds the number of score-element pairs. */

    /* Check for incompatible options. */
    //nx和xxflag互斥,二者不能同时出现
    if (nx && xx) {
        addReplyError(c,
            "XX and NX options at the same time are not compatible");
        return;
    }
    //若有incr标志,则只能有一对<element,score>
    //为什么不能是多对?
    if (incr && elements > 1) {
        addReplyError(c,
            "INCR option supports a single increment-element pair");
        return;
    }

    /* Start parsing all the scores, we need to emit any syntax error
     * before executing additions to the sorted set, as the command should
     * either execute fully or nothing at all. */
    //依次检查每一个分数值
    scores = zmalloc(sizeof(double)*elements);
    for (j = 0; j < elements; j++) {
        //该函数中会检查score是否是合法的double类型的值
        if (getDoubleFromObjectOrReply(c,c->argv[scoreidx+j*2],&scores[j],NULL)
            != C_OK) goto cleanup;
    }

    /* Lookup the key and create the sorted set if does not exist. */
    //根据key查找对应的有序集合的value
    zobj = lookupKeyWrite(c->db,key);
    //key不存在
    if (zobj == NULL) {
        //如果设置了xx这个flag,直接返回错误
        if (xx) goto reply_to_client; /* No key + XX option: nothing to do. */
        //根据redis的配置,如果有序集合设置了不使用ziplist存储或者说第一个插入元素的长度大于
        //设置的最大ziplist的元素长度值,则使用跳跃表存储否则使用ziplist
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
        {
            zobj = createZsetObject();
        } else {
            zobj = createZsetZiplistObject();
        }
        //把key,zobj插入字典
        dbAdd(c->db,key,zobj);
    //key存在
    } else {
        //如果不是有序集合,直接返回错误
        if (zobj->type != OBJ_ZSET) {
            addReply(c,shared.wrongtypeerr);
            goto cleanup;
        }
    }

    //elements是<member,score>对数
    for (j = 0; j < elements; j++) {
        double newscore;
        score = scores[j];
        //retflags设置为前文中的flags变量
        int retflags = flags;

        ele = c->argv[scoreidx+1+j*2]->ptr;
        //每次遍历,score是分数,ele是member.调用zsetadd插入zobj
        int retval = zsetAdd(zobj, score, ele, &retflags, &newscore);
        if (retval == 0) {
            addReplyError(c,nanerr);
            goto cleanup;
        }
        //根据retflags,即一个元素是更新还是新加入,还是未做处理(即member存在,并且
        //score值与新设置的一致),更新相应的计数变量(这些变量最后会返回给客户端)
        if (retflags & ZADD_ADDED) added++;
        if (retflags & ZADD_UPDATED) updated++;
        if (!(retflags & ZADD_NOP)) processed++;
        score = newscore;
    }
    server.dirty += (added+updated);
//通过命令中的flag,返回给客户端不同的值
reply_to_client:
    if (incr) { /* ZINCRBY or INCR option. */
        if (processed)
            addReplyDouble(c,score);
        else
            addReply(c,shared.nullbulk);
    } else { /* ZADD. */
        addReplyLongLong(c,ch ? added+updated : added);
    }

//如果有更新或者新加,需要执行相应的watch key的通知及keyspace的通知
cleanup:
    zfree(scores);
    if (added || updated) {
        signalModifiedKey(c->db,key);
        notifyKeyspaceEvent(NOTIFY_ZSET,
            incr ? "zincr" : "zadd", key, c->db->id);
    }
}
```

## ZINCRBY 

ZINCRBY key increment member

如果key存在,就给相应member的score增加increment

否则直接给key设置分数为increment

```c

//与zadd调用同一个函数,相当于zadd key incr,把incr flag置位
void zincrbyCommand(client *c) {
    zaddGenericCommand(c,ZADD_INCR);
}
```

## ZCARD
ZCARD key

返回有序集合的元素个数
```c
void zcardCommand(client *c) {
    robj *key = c->argv[1];
    robj *zobj;
    //查找key对应的value
    if ((zobj = lookupKeyReadOrReply(c,key,shared.czero)) == NULL ||
        checkType(c,zobj,OBJ_ZSET)) return;
    //通过zsetLength获取zobj中的元素个数
    addReplyLongLong(c,zsetLength(zobj));
}
```

```c
unsigned int zsetLength(const robj *zobj) {
    int length = -1;
    //如果是ziplist,通过zzlLength函数获取长度
    //如果长度字段中的值小于UINT16_MAX，直接返回长度。否则需要遍历获取长度
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        length = zzlLength(zobj->ptr);
    //如果是skiplist,直接返回zsl->length
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        length = ((const zset*)zobj->ptr)->zsl->length;
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return length;
}
```

## ZCOUNT
ZCOUNT key min max

返回key中score值在min和max之间的元素个数

其中min和max可以加(,如 zcount key (5 (10 

加左括号表示不包含。不加表示包含

```c
void zcountCommand(client *c) {
    robj *key = c->argv[1];
    robj *zobj;
    zrangespec range;
    int count = 0;

    /* Parse the range arguments */
    //判定范围.并将最大最小及是否包含写入range结构体中
    if (zslParseRange(c->argv[2],c->argv[3],&range) != C_OK) {
        addReplyError(c,"min or max is not a float");
        return;
    }

    /* Lookup the sorted set */
    if ((zobj = lookupKeyReadOrReply(c, key, shared.czero)) == NULL ||
        checkType(c, zobj, OBJ_ZSET)) return;
    //判断zobj底层编码是ziplist还是skiplist
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl = zobj->ptr;
        unsigned char *eptr, *sptr;
        double score;
        //找出第一个在范围之内的元素
        /* Use the first element in range as the starting point */
        eptr = zzlFirstInRange(zl,&range);

        /* No "first" element */
        if (eptr == NULL) {
            addReply(c, shared.czero);
            return;
        }

        /* First element is in range */
        //ziplist中member和score是两个entry,并且member之后保存着score
        //整体顺序是按score从小到大排列,score相同时,按member的字典序排列
        sptr = ziplistNext(zl,eptr);
        //所以此处从第一个元素的下一个entry处获取score
        score = zzlGetScore(sptr);
        serverAssertWithInfo(c,zobj,zslValueLteMax(score,&range));

        /* Iterate over elements in range */
        //迭代这个ziplist,如果score满足要求,则count++并且继续迭代,否则跳出
        //最后会返回count
        while (eptr) {
            score = zzlGetScore(sptr);

            /* Abort when the node is no longer in range. */
            if (!zslValueLteMax(score,&range)) {
                break;
            } else {
                count++;
                zzlNext(zl,&eptr,&sptr);
            }
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        zskiplistNode *zn;
        unsigned long rank;
        //如果是跳表,也是先取出第一个元素
        /* Find first element in range */
        zn = zslFirstInRange(zsl, &range);

        /* Use rank of first element, if any, to determine preliminary count */
        if (zn != NULL) {
            //获取第一个元素的排名
            rank = zslGetRank(zsl, zn->score, zn->ele);
            count = (zsl->length - (rank - 1));
            //如果最大值大于zsl中的最大值,则此count就是要找的个数
            /* Find last element in range */
            zn = zslLastInRange(zsl, &range);

            /* Use rank of last element, if any, to determine the actual count */
            if (zn != NULL) {
                //如果最大值小于zsl中的最大值，则首先找到最后一个元素的rank
                rank = zslGetRank(zsl, zn->score, zn->ele);
                //重新计算count,与之前的计算公式合并之后为
                //count = (zsl->length-(rankmin-1))-(zsl->length-rankmax))
                //      = rankmax-rankmin+1
                count -= (zsl->length - rank);
            }
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    //返回count
    addReplyLongLong(c, count);
}
```

## ZRANGEBYSCORE
ZRANGEBYSCORE key min max [WITHSCORES][LIMIT offset count]

获取有序结合中分值位于 min和max之间的所有元素

withscores:将member 和 score一起返回 

limit offset count:从偏移offset开始获取count个元素

min和max可以为 -inf,+inf,分别表示负无穷和正无穷

```c

//入口函数
void zrangebyscoreCommand(client *c) {
    genericZrangebyscoreCommand(c,0);
}

```

```c
/* This command implements ZRANGEBYSCORE, ZREVRANGEBYSCORE. */
void genericZrangebyscoreCommand(client *c, int reverse) {
    zrangespec range;
    robj *key = c->argv[1];
    robj *zobj;
    long offset = 0, limit = -1;
    int withscores = 0;
    unsigned long rangelen = 0;
    void *replylen = NULL;
    int minidx, maxidx;
    //该函数同时用于zrangbyscore和zrevrangebyscore
    //二者通过函数中的reverse参数标识
    //正序时第二个参数是min，第三个参数是max,逆序反之
    /* Parse the range arguments. */
    if (reverse) {
        /* Range is given as [max,min] */
        maxidx = 2; minidx = 3;
    } else {
        /* Range is given as [min,max] */
        minidx = 2; maxidx = 3;
    }
    //将参数解析出来赋值到range变量
    if (zslParseRange(c->argv[minidx],c->argv[maxidx],&range) != C_OK) {
        addReplyError(c,"min or max is not a float");
        return;
    }

    /* Parse optional extra arguments. Note that ZCOUNT will exactly have
     * 4 arguments, so we'll never enter the following code path. */
    if (c->argc > 4) {
        int remaining = c->argc - 4;
        int pos = 4;
        //解析withscores和limit参数
        while (remaining) {
            if (remaining >= 1 && !strcasecmp(c->argv[pos]->ptr,"withscores")) {
                pos++; remaining--;
                withscores = 1;
            } else if (remaining >= 3 && !strcasecmp(c->argv[pos]->ptr,"limit")) {
                if ((getLongFromObjectOrReply(c, c->argv[pos+1], &offset, NULL)
                        != C_OK) ||
                    (getLongFromObjectOrReply(c, c->argv[pos+2], &limit, NULL)
                        != C_OK))
                {
                    return;
                }
                pos += 3; remaining -= 3;
            } else {
                addReply(c,shared.syntaxerr);
                return;
            }
        }
    }

    /* Ok, lookup the key and get the range */
    if ((zobj = lookupKeyReadOrReply(c,key,shared.emptymultibulk)) == NULL ||
        checkType(c,zobj,OBJ_ZSET)) return;
    //按zset底层编码是ziplist还是skiplist分别处理
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl = zobj->ptr;
        unsigned char *eptr, *sptr;
        unsigned char *vstr;
        unsigned int vlen;
        long long vlong;
        double score;

        /* If reversed, get the last node in range as starting point. */
        //按正序还是逆序分别取最后一个值或者第一个值
        if (reverse) {
            eptr = zzlLastInRange(zl,&range);
        } else {
            eptr = zzlFirstInRange(zl,&range);
        }

        /* No "first" element in the specified interval. */
        if (eptr == NULL) {
            addReply(c, shared.emptymultibulk);
            return;
        }

        /* Get score pointer for the first element. */
        serverAssertWithInfo(c,zobj,eptr != NULL);
        sptr = ziplistNext(zl,eptr);

        /* We don't know in advance how many matching elements there are in the
         * list, so we push this object that will represent the multi-bulk
         * length in the output buffer, and will "fix" it later */
        replylen = addDeferredMultiBulkLength(c);

        /* If there is an offset, just traverse the number of elements without
         * checking the score because that is done in the next loop. */
        //如果有offset,先偏移相应的元素
        //注意此处zzlNext传入了两个指针,会一次偏移一个<member,score>对
        //注意此处offset初始值是0,如果没指定则不会进入此处循环
        while (eptr && offset--) {
            if (reverse) {
                zzlPrev(zl,&eptr,&sptr);
            } else {
                zzlNext(zl,&eptr,&sptr);
            }
        }

        //如果有limit,则进入循环.取limit次.limit的初始值为-1,即使没指定,也会进入循环
        //直到eptr为null或者循环中break掉
        while (eptr && limit--) {
            score = zzlGetScore(sptr);

            /* Abort when the node is no longer in range. */
            //不在范围之内时break掉
            if (reverse) {
                if (!zslValueGteMin(score,&range)) break;
            } else {
                if (!zslValueLteMax(score,&range)) break;
            }

            /* We know the element exists, so ziplistGet should always succeed */
            serverAssertWithInfo(c,zobj,ziplistGet(eptr,&vstr,&vlen,&vlong));
            //取出相应的值.可能为str,赋值给vstr,长度为vlen,或者为整型,赋值给vlong
            rangelen++;
            if (vstr == NULL) {
                addReplyBulkLongLong(c,vlong);
            } else {
                addReplyBulkCBuffer(c,vstr,vlen);
            }
            //如果设置了withscores标志,则返回分数
            if (withscores) {
                addReplyDouble(c,score);
            }
            //开始迭代下一个节点
            /* Move to next node */
            if (reverse) {
                zzlPrev(zl,&eptr,&sptr);
            } else {
                zzlNext(zl,&eptr,&sptr);
            }
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        zskiplistNode *ln;

        //同ziplist,先查找起始或最终节点
        /* If reversed, get the last node in range as starting point. */
        if (reverse) {
            ln = zslLastInRange(zsl,&range);
        } else {
            ln = zslFirstInRange(zsl,&range);
        }

        /* No "first" element in the specified interval. */
        if (ln == NULL) {
            addReply(c, shared.emptymultibulk);
            return;
        }

        /* We don't know in advance how many matching elements there are in the
         * list, so we push this object that will represent the multi-bulk
         * length in the output buffer, and will "fix" it later */
        //返回客户端时先返回元素个数,但此处并不知道需要返回多少个元素,所以先占个位置
        //replylen是存储len字段的指针
        replylen = addDeferredMultiBulkLength(c);

        /* If there is an offset, just traverse the number of elements without
         * checking the score because that is done in the next loop. */
        //处理offset.向前或向后skip
        while (ln && offset--) {
            if (reverse) {
                ln = ln->backward;
            } else {
                ln = ln->level[0].forward;
            }
        }

        //处理limit
        while (ln && limit--) {
            /* Abort when the node is no longer in range. */
            if (reverse) {
                if (!zslValueGteMin(ln->score,&range)) break;
            } else {
                if (!zslValueLteMax(ln->score,&range)) break;
            }

            rangelen++;
            addReplyBulkCBuffer(c,ln->ele,sdslen(ln->ele));

            if (withscores) {
                addReplyDouble(c,ln->score);
            }

            /* Move to next node */
            if (reverse) {
                ln = ln->backward;
            } else {
                ln = ln->level[0].forward;
            }
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    //如果有withscores参数,返回给客户端的字符串数量是2倍
    if (withscores) {
        rangelen *= 2;
    }
    //将rangelen放入replylen指向的位置,返回给客户端
    setDeferredMultiBulkLength(c, replylen, rangelen);
}
```

## ZRANK
ZRANK key member

返回有序集合中元素member的rank

以0为起始rank,元素分数从低到高

zrevrank,元素分数从高到低

```c
void zrankGenericCommand(client *c, int reverse) {
    robj *key = c->argv[1];
    robj *ele = c->argv[2];
    robj *zobj;
    long rank;
    //通过key找出有序集合的value zobj
    if ((zobj = lookupKeyReadOrReply(c,key,shared.nullbulk)) == NULL ||
        checkType(c,zobj,OBJ_ZSET)) return;

    serverAssertWithInfo(c,ele,sdsEncodedObject(ele));
    //在zobj中查找ele(第二个参数member)
    rank = zsetRank(zobj,ele->ptr,reverse);
    if (rank >= 0) {
        addReplyLongLong(c,rank);
    } else {
        addReply(c,shared.nullbulk);
    }
}

void zrankCommand(client *c) {
    zrankGenericCommand(c, 0);
}
```
```c
long zsetRank(robj *zobj, sds ele, int reverse) {
    unsigned long llen;
    unsigned long rank;

    llen = zsetLength(zobj);
    //ziplist从前往后遍历,比较entry中的元素与ele,每次将rank++
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl = zobj->ptr;
        unsigned char *eptr, *sptr;

        eptr = ziplistIndex(zl,0);
        serverAssert(eptr != NULL);
        sptr = ziplistNext(zl,eptr);
        serverAssert(sptr != NULL);

        rank = 1;
        while(eptr != NULL) {
            if (ziplistCompare(eptr,(unsigned char*)ele,sdslen(ele)))
                break;
            rank++;
            zzlNext(zl,&eptr,&sptr);
        }

        if (eptr != NULL) {
            //如果是逆序取,直接将llen-rank就是逆向的rank
            if (reverse)
                return llen-rank;
            else
                return rank-1;
        } else {
            return -1;
        }
    //skiplist通过zslGetRank获取rank,具体过程为跳表查找,将相应路过节点的span相加
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        dictEntry *de;
        double score;

        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            score = *(double*)dictGetVal(de);
            rank = zslGetRank(zsl,score,ele);
            /* Existing elements always have a rank. */
            serverAssert(rank != 0);
            //逆向取rank
            if (reverse)
                return llen-rank;
            else
                return rank-1;
        } else {
            return -1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
}
```

## ZREM
ZREM key member [member ...]

从有序集合中删除相应的member


```c
void zremCommand(client *c) {
    robj *key = c->argv[1];
    robj *zobj;
    int deleted = 0, keyremoved = 0, j;
    //根据key找到对应的zobj
    if ((zobj = lookupKeyWriteOrReply(c,key,shared.czero)) == NULL ||
        checkType(c,zobj,OBJ_ZSET)) return;
    //依次删除相应的元素,每次删除之后检查zset是否为空,如果为空,删掉该key,并且break
    for (j = 2; j < c->argc; j++) {
        if (zsetDel(zobj,c->argv[j]->ptr)) deleted++;
        if (zsetLength(zobj) == 0) {
            dbDelete(c->db,key);
            keyremoved = 1;
            break;
        }
    }
    //如果确实有member被删除掉,通知keyspace zrem事件
    //如果zset整个都被删除了,通知keyspace del事件
    if (deleted) {
        notifyKeyspaceEvent(NOTIFY_ZSET,"zrem",key,c->db->id);
        if (keyremoved)
            notifyKeyspaceEvent(NOTIFY_GENERIC,"del",key,c->db->id);
        signalModifiedKey(c->db,key);
        server.dirty += deleted;
    }
    //返回给客户端实际删除的member个数
    addReplyLongLong(c,deleted);
}
```

```c
/* Delete the element 'ele' from the sorted set, returning 1 if the element
 * existed and was deleted, 0 otherwise (the element was not there). */
int zsetDel(robj *zobj, sds ele) {
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *eptr;
        //ziplist先找到ele所在位置的指针eptr
        if ((eptr = zzlFind(zobj->ptr,ele,NULL)) != NULL) {
            //将该元素删除.ziplist删除时会resize,此处将删除之后ziplist的指针复值给zobj->ptr
            zobj->ptr = zzlDelete(zobj->ptr,eptr);
            return 1;
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        dictEntry *de;
        double score;
        //skiplist现将zobj->ptr->dict相应的ele删除掉。此处并未真实删除
        //而是将ele所在的dictEntry返回
        de = dictUnlink(zs->dict,ele);
        if (de != NULL) {
            /* Get the score in order to delete from the skiplist later. */
            //通过dictEntry获取score
            score = *(double*)dictGetVal(de);

            /* Delete from the hash table and later from the skiplist.
             * Note that the order is important: deleting from the skiplist
             * actually releases the SDS string representing the element,
             * which is shared between the skiplist and the hash table, so
             * we need to delete from the skiplist as the final step. */
            //此处将dict中的key和value实际free掉
            dictFreeUnlinkedEntry(zs->dict,de);

            /* Delete from skiplist. */
            //从skiplist中删除元素.ele这个sds在hash和skiplist共享.从skiplist中删除时
            //会释放此sds,所以必须先删除dict中的元素再删除skiplist中的元素
            int retval = zslDelete(zs->zsl,score,ele,NULL);
            serverAssert(retval);
            //如果hash表中元素使用率小于10%,进行dict的resize
            if (htNeedsResize(zs->dict)) dictResize(zs->dict);
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* No such element found. */
}
```