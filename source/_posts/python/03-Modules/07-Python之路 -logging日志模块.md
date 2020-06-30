---
title:  22.Python之路 - logging日志模块
date: 2020-06-26 09:20:58
tags:  python
categories: [python,03-Modules]
comments: true
copyright: true
---



# Python之路 -  logging日志模块

##  介绍 🍀

logging模块是Python内置的标准模块，主要用于输出运行日志，可以设置输出日志的等级、日志保存路径、日志文件回滚等；相比print，具备如下优点：

可以通过设置不同的日志等级，在release版本中只输出重要信息，而不必显示大量的调试信息；

print将所有信息都输出到标准输出中，严重影响开发者从标准输出中查看其它数据；logging则可以由开发者决定将信息输出到什么地方，以及怎么输出。

<!--more-->


## logging模块日志等级 🍀
logging模块默认定义了以下几个日志等级，它允许开发人员自定义其他日志级别，但是这是不被推荐的，尤其是在开发供别人使用的库时，因为这会导致日志级别的混乱。


|   日志等级（level）   |   描述   |
|     ---- | ---- |
|   DEBUG   | 详细信息，典型地调试问题时会感兴趣。 详细的debug信息。     |
|    INFO  |    证明事情按预期工作。 关键事件。  |
|  WARNING    |    表明发生了一些意外，或者不久的将来会发生问题（如‘磁盘满了’）。软件还是在正常工作。  |
|   ERROR   | 由于更严重的问题，软件已不能执行一些功能了。 一般错误消息。     |
|   CRITICAL   |  严重错误，表明软件已不能继续运行了。    |


logging模块定义的模块级别的常用函数

|   函数   |   说明   |
| ---- | ---- |
|   logging.debug(msg, *args, **kwargs)	   |   创建一条严重级别为DEBUG的日志记录   |
|  logging.info(msg, *args, **kwargs)    |   创建一条严重级别为INFO的日志记录   |
|   logging.warning(msg, *args, **kwargs)   |  创建一条严重级别为WARNING的日志记录    |
|   logging.error(msg, *args, **kwargs)   |   	创建一条严重级别为ERROR的日志记录   |
|   logging.critical(msg, *args, **kwargs)   |  创建一条严重级别为CRITICAL的日志记录    |
|  logging.log(level, *args, **kwargs)    |   创建一条严重级别为level的日志记录   |
|   logging.basicConfig(**kwargs)   |  对root logger进行一次性配置    |

## 一.logging模块的基础使用方法 🍀

看下面的简单例子:

```
import logging
logging.debug("debug_msg")
logging.info("info_msg")
logging.warning("warning_msg")
logging.error("error_msg")
logging.critical("critical_msg")

>>> 执行结果:
WARNING:root:warning_msg
ERROR:root:error_msg
CRITICAL:root:critical_msg
```
默认情况下Python的logging模块将日志打印到了标准输出中，且只显示了大于等于WARNING级别的日志，这说明默认的日志级别设置为WARNING（日志级别等级CRITICAL > ERROR > WARNING > INFO > DEBUG）

默认输出格式为

```
默认的日志格式为日志级别：Logger名称：用户输出消息
```

 logging.basicConfig()函数调整日志级别、输出格式等

```
 logging.basicConfig(level=logging.DEBUG,
                    format="%(asctime)s %(name)s %(levelname)s %(message)s",
                    datefmt = '%Y-%m-%d  %H:%M:%S',
                    filename='sys.log'
                    )

logging.debug("debug调试")
logging.info("正常日志信息")
logging.warning("警告信息")
logging.error("错误信息")
logging.critical("严重错误")
                  
```
执行后,查看sys.log文件,格式化的记录了日志信息:

```
2019-06-15  16:20:21 root DEBUG debug调试
2019-06-15  16:20:21 root INFO 正常日志信息
2019-06-15  16:20:21 root WARNING 警告信息
2019-06-15  16:20:21 root ERROR 错误信息
2019-06-15  16:20:21 root CRITICAL 严重错误
```

basicConfig具体参数说明:

```
logging.basicConfig()函数中可通过具体参数来更改logging模块默认行为，可用参数有：

filename：  用指定的文件名创建FiledHandler，这样日志会被存储在指定的文件中。
filemode：  文件打开方式，在指定了filename时使用这个参数，默认值为“a”还可指定为“w”。
format：    指定handler使用的日志显示格式。
datefmt：   指定日期时间格式。
level：     设置rootlogger（后边会讲解具体概念）的日志级别
stream：    用指定的stream创建StreamHandler。可以指定输出到sys.stderr,sys.stdout或者文件(f=open(‘test.log’,’w’))，默认为sys.stderr。若同时列出了filename和stream两个参数，则stream参数会被忽略。


format参数中可能用到的格式化串：
%(name)s Logger的名字
%(levelno)s 数字形式的日志级别
%(levelname)s 文本形式的日志级别
%(pathname)s 调用日志输出函数的模块的完整路径名，可能没有
%(filename)s 调用日志输出函数的模块的文件名
%(module)s 调用日志输出函数的模块名
%(funcName)s 调用日志输出函数的函数名
%(lineno)d 调用日志输出函数的语句所在的代码行
%(created)f 当前时间，用UNIX标准的表示时间的浮 点数表示
%(relativeCreated)d 输出日志信息时的，自Logger创建以 来的毫秒数
%(asctime)s 字符串形式的当前时间。默认格式是 “2003-07-08 16:49:45,896”。逗号后面的是毫秒
%(thread)d 线程ID。可能没有
%(threadName)s 线程名。可能没有
%(process)d 进程ID。可能没有
%(message)s用户输出的消息
```



## 二.logging模块的进阶使用方法 🍀

第二种是一个日志流处理流程,通过函数logging.getLogger([name])（返回一个logger对象，如果没有指定名字将返回root logger）。


### logging日志模块四大组件
在介绍logging模块的日志流处理流程之前，我们先来介绍下logging模块的四大组件：

| 组件名称 | 对应类名  | **功能描述**                                                 |
| -------- | --------- | ------------------------------------------------------------ |
| 日志器   | Logger    | 提供了应用程序可一直使用的接口                               |
| 处理器   | Handler   | 将logger创建的日志记录发送到合适的目的输出                   |
| 过滤器   | Filter    | 提供了更细粒度的控制工具来决定输出哪条日志记录，丢弃哪条日志记录 |
| 格式器   | Formatter | 决定日志记录的最终输出格式                                   |

###  这些组件之间的关系描述：

日志器（logger）需要通过处理器（handler）将日志信息输出到目标位置，如：文件、sys.stdout、网络等；

不同的处理器（handler）可以将日志输出到不同的位置；

日志器（logger）可以设置多个处理器（handler）将同一条日志记录输出到不同的位置；

每个处理器（handler）都可以设置自己的过滤器（filter）实现日志过滤，从而只保留感兴趣的日志；

每个处理器（handler）都可以设置自己的格式器（formatter）实现同一条日志以不同的格式输出到不同的地方。

简单点说就是：日志器（logger）是入口，真正干活儿的是处理器（handler），处理器（handler）还可以通过过滤器（filter）和格式器（formatter）对要输出的日志内容做过滤和格式化等处理操作。

### 日志流处理简要流程

1、创建一个logger

2、设置下logger的日志的等级

3、创建合适的Handler(FileHandler要有路径)

4、设置下每个Handler的日志等级

5、创建下日志的格式

6、向Handler中添加上面创建的格式

7、将上面创建的Handler添加到logger中

8、打印输出logger.debug\logger.info\logger.warning\logger.error\logger.critical


###下面是一个例子  🍀

```
import logging

#step1: 创建logger对象.如果参数为空,则返回root logger
logger = logging.getLogger()

#step2: 设置logger日志等级.可以是级别名,还可以是数字

logger.setLevel(logging.DEBUG)

#step3: 创建输出流对象

#创建handler文件句柄对象
fh = logging.FileHandler('test.log',encoding='utf-8')
#创建控制台输出对象
sh = logging.StreamHandler()

#step4.定义日志格式

formatter = logging.Formatter(
    fmt="%(asctime)s %(name)s %(filename)s %(message)s",
    datefmt="%Y/%m/%d %X"
    )


#step5.输出流对象和日志格式进行关联.格式化日志输出

fh.setFormatter(formatter)
sh.setFormatter(formatter)

#step6. 添加输出流对象和logger对象

logger.addHandler(fh)
logger.addHandler(sh)

#step7: 输出日志

logger.debug("debug调试")
logger.info("正常日志信息")
logger.warning("警告信息")
logger.error("错误信息")
logger.critical("严重错误")

>>> 执行结果:
2019/06/15 16:26:20 root python06.logging模块.py debug调试
2019/06/15 16:26:20 root python06.logging模块.py 正常日志信息
2019/06/15 16:26:20 root python06.logging模块.py 警告信息
2019/06/15 16:26:20 root python06.logging模块.py 错误信息
2019/06/15 16:26:20 root python06.logging模块.py 严重错误
```

python logging 重复写日志问题
用Python的logging模块记录日志时，可能会遇到重复记录日志的问题，第一条记录写一次，第二条记录写两次，第三条记录写三次

稍微修改上面代码,通过函数调用:

```
import logging

def log(msg):
    logger = logging.getLogger()
    logger.setLevel(logging.DEBUG)

    fh = logging.FileHandler('test.log',encoding='utf-8')
    sh = logging.StreamHandler()

    formatter = logging.Formatter(
        fmt="%(asctime)s %(name)s %(filename)s %(message)s",
        datefmt="%Y/%m/%d %X"
        )


    fh.setFormatter(formatter)
    sh.setFormatter(formatter)

    logger.addHandler(fh)
    logger.addHandler(sh)

    logger.info(msg)

#解决重复打印日志方法一:每次调用完就关闭文件句柄
    # logger.removeHandler(fh)
    # logger.removeHandler(sh)



log("前方注意")
log("提示")
log("错误")

>>> 打印结果:
2019/06/15 16:28:20 root python06.logging模块.py 前方注意
2019/06/15 16:28:20 root python06.logging模块.py 提示
2019/06/15 16:28:20 root python06.logging模块.py 提示
2019/06/15 16:28:20 root python06.logging模块.py 错误
2019/06/15 16:28:20 root python06.logging模块.py 错误
2019/06/15 16:28:20 root python06.logging模块.py 错误
```
> 重复打印原因：第二次调用log的时候，根据getLogger(name)里的name获取同一个logger，
> 
>而这个logger里已经有了第一次你添加的handler，第二次调用又添加了一个handler，
>
>所以，这个logger里有了两个同样的handler，以此类推，调用几次就会有几个handler。

第二种解决方法.每次创建文件句柄对象时,先判断是否已经存在一个对象

```
import logging

def log(msg):
    logger = logging.getLogger()
    logger.setLevel(logging.DEBUG)

    #解决方案2.handlers对象,如果没有就创建.否则直接记录日志
    if not logger.handlers:
        fh = logging.FileHandler('test.log',encoding='utf-8')
        sh = logging.StreamHandler()

        formatter = logging.Formatter(
            fmt="%(asctime)s %(name)s %(filename)s %(message)s",
            datefmt="%Y/%m/%d %X"
            )


        fh.setFormatter(formatter)
        sh.setFormatter(formatter)

        logger.addHandler(fh)
        logger.addHandler(sh)

    logger.info(msg)




log("前方注意")
log("提示")
log("错误")

```

一个完整的使用例子:

```
import logging

def log():
    logger = logging.getLogger()
    logger.setLevel(logging.DEBUG)

    #解决方案2.handlers对象,如果没有就创建.否则直接记录日志
    if not logger.handlers:
        fh = logging.FileHandler('test.log',encoding='utf-8')
        sh = logging.StreamHandler()

        formatter = logging.Formatter(
            fmt="%(asctime)s %(name)s %(filename)s %(message)s",
            datefmt="%Y/%m/%d %X"
            )


        fh.setFormatter(formatter)
        sh.setFormatter(formatter)

        logger.addHandler(fh)
        logger.addHandler(sh)

    return logger




log_record = log()
log_record.info('前方注意')
log_record.error('前方报错')
log_record.debug('前方正常')
```

## 三.logging模块的高级使用方法 🍀


将上述配置定义在一个logging字典中.然后用该字典实例化一个logger对象.

```
import os
import logging.config

# 定义三种日志输出格式 开始

standard_format = '[%(asctime)s][%(threadName)s:%(thread)d][task_id:%(name)s][%(filename)s:%(lineno)d]' \
                  '[%(levelname)s][%(message)s]' #其中name为getlogger指定的名字

simple_format = '[%(levelname)s][%(asctime)s][%(filename)s:%(lineno)d]%(message)s'

id_simple_format = '[%(levelname)s][%(asctime)s] %(message)s'

# 定义日志输出格式 结束

logfile_dir = os.path.dirname(os.path.abspath(__file__))  # log文件的目录

logfile_name = 'advance.log'  # log文件名

# log文件的全路径
logfile_path = os.path.join(logfile_dir, logfile_name)

# log配置字典
LOGGING_DIC = {
    'version': 1,
    'disable_existing_loggers': False,
    #定义2个日志格式
    'formatters': {
        'standard': {
            'format': standard_format
        },
        'simple': {
            'format': simple_format
        },
    },
    'filters': {},
    'handlers': {
        #打印到终端的日志
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',  # 打印到屏幕
            'formatter': 'simple'
        },
        #打印到文件的日志,收集info及以上的日志
        'default': {
            'level': 'DEBUG',
            'class': 'logging.handlers.RotatingFileHandler',  # 保存到文件
            'formatter': 'standard',
            'filename': logfile_path,  # 日志文件
            'maxBytes': 1024*1024*5,  # 日志大小 5M
            'backupCount': 5,
            'encoding': 'utf-8',  # 日志文件的编码，再也不用担心中文log乱码了
        },
    },
    'loggers': {
        #logging.getLogger(__name__)拿到的logger配置
        '': {
            'handlers': ['default', 'console'],  # 这里把上面定义的两个handler都加上，即log数据既写入文件又打印到屏幕
            'level': 'DEBUG',
            'propagate': True,  # 向上（更高level的logger）传递
        },
    },
}


def load_my_logging_cfg():
    logging.config.dictConfig(LOGGING_DIC)  # 导入上面定义的logging配置
    logger = logging.getLogger(__name__)  # 生成一个log实例
    return logger

```

实例化对象,记录日志:

```
log = load_my_logging_cfg()
log.info("程序正常")
```

执行结果:

```
[INFO][2019-06-15 16:33:40,852][python06.logging模块.py:271]程序正常

#advance.log日志输出:
[2019-06-15 16:33:40,852][MainThread:11892][task_id:__main__][python06.logging模块.py:271][INFO][程序正常]
```

有了上述方式我们的好处是：所有与logging模块有关的配置都写到字典中就可以了，更加清晰，方便管理


关于上面字典中的Loggers键子字典中的key为什么为空.?
```
 logger对象都是配置到字典的loggers 键对应的子字典中的
    按照我们对logging模块的理解，要想获取某个东西都是通过名字，也就是key来获取的
    于是我们要获取不同的logger对象就是
    logger=logging.getLogger('loggers子字典的key名')

    
    但问题是：如果我们想要不同logger名的logger对象都共用一段配置，那么肯定不能在loggers子字典中定义n个key   
 'loggers': {    
        'l1': {
            'handlers': ['default', 'console'],  #
            'level': 'DEBUG',
            'propagate': True,  # 向上（更高level的logger）传递
        },
        'l2: {
            'handlers': ['default', 'console' ], 
            'level': 'DEBUG',
            'propagate': False,  # 向上（更高level的logger）传递
        },
        'l3': {
            'handlers': ['default', 'console'],  #
            'level': 'DEBUG',
            'propagate': True,  # 向上（更高level的logger）传递
        },

}

    
#我们的解决方式是，定义一个空的key
    'loggers': {
        '': {
            'handlers': ['default', 'console'], 
            'level': 'DEBUG',
            'propagate': True, 
        },

}

这样我们再取logger对象时
logging.getLogger(__name__)，不同的文件__name__不同，这保证了打印日志时标识信息不同，但是拿着该名字去loggers里找key名时却发现找不到，于是默认使用key=''的配置
```