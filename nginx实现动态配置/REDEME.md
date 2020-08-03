## nginx实现动态配置
### 背景
传统的nginx配置，需要配置静态的nginx server.conf, 然后发布到线上，之后对nginx进行reload。这在大规模的商业服务中，是很难接受的。

主要问题如下:
- nginx server.conf只是一些指令配置, 无法跟实际业务及功能联动，这需要有专门的运维同学负责将相关业务需求转化为对应的nginx server.conf，并进行功能验证。
- nginx reload/restart成本太高。对一台qps为几十K以上的nginx服务进行reload, 相信任何的运维同学都比较担忧。虽然说nginx reload是服务无损的，但是一定是这样吗？
- 像阿里云、金山云商业CDN，光接入的域名就多达上百万，如果将这上百万的配置转换成nginx server.conf，一个是管理比较麻烦，另一个也不利于nginx配置加载。

我们以10W个域名为例，每个域名的配置为1K， 光读取并解析100M的文件就需要10s以上，这在商业应用中，很显然是不可接受的。

随着业务的快速发展，如果上述工作全部交给运维同学去保障，无疑要消耗大量的人力物力，同时还要面临管理混乱，因配置变更、服务重启导致的各种异常情况, 因此接入标准化势在必行。


### Nginx动态配置实现方案
#### 在nginx原生的配置系统上实现server.conf reload
原nginx配置更新，需要先启动一批新的worker, 然后关闭旧worker进程的socket fd。让新worker接受新的请求，而旧woker不再接受任何请求，等待旧的worker处理完遗留的任务后退出，即可实现nginx的平滑重启。

而该方案是在不重启nginx的情况化，在进程内部重新对nginx server.conf进行解析，并对原生的nginx http conf对象进行替换，从而达到配置热加载的效果。

优缺点:
- 复用nginx原生的功能和使用习惯，不存在性能等问题。
- 缺点是运维人员需要将相关功能转换为对应的nginx配置，或者是根据线上业务，转化为所需要的配置。

实现方式参考include指令即可, 内部细节略有不同（数据结构组织）.
```
仿照ngx_conf_parse(cf, &file)对server conf进行解析，并将对应的conf对象保存到自定义的table中。
仿照ngx_http_find_virtual_server重写server conf查找逻辑。原本是从port管理的server table中进行查找，现在改成我们自定义的table即可。
```

#### 使用openresty重构nginx业务逻辑
在该方案中，我们会创建一个基础的nginx server conf, 由它接受请求，并将后续的请求控制逻辑全部交给lua来处理。

优缺点:
- 需要在lua中重新实现nginx大部分功能，主要如多path、rewrite、限流限速等
- lua性能比之c还是存在一定的差距，随着功能越来越复杂，服务优化的工作会越来越重。
- 过往经验中，lua虽然简单易用，但是lua使用的比较好的开发人员却比较少，这些开发人员通常会给项目埋很多的坑，并且会引入不少的性能问题。
- 有些业务方期望对连接层、协议层进行优化，这种情况下，lua表现的非常乏力。如调整某个域名的请求的超时控制、recv buffer和send buffer等。

针对lua对连接参数、协议参数设置的不足，我们会对nginx原生代码进行修改，使之其变成可以在lua中控制的功能。

大体实现方案有两种:
- nginx配置变量化:
    - 将nginx配置的value修改成可使用nginx变量的方式，在lua中修改这些nginx变量以达到动态配置的目的。
    - 对nginx代码侵入较大，过多的nginx变量可能会存在性能问题。
- 更改value获取逻辑:
    - 不改动原生nginx配置对象的内容，而是在配置查找过程中，转入我们定义好的扩展配置中。
    - 比如原生Nginx的http超时参数需要从server conf中获取，而我们提供一个http_timeout_get的函数，在该函数的内容先查询我们自己的配置，并返回，否则以原本的查找流程进行兜底。
    - 对nginx代码入侵较大，该方式会导致nginx代码变的难以阅读。

```
更改value的获取逻辑:
针对每处需要修改的地方进行封装, 如下 (伪代码)：
u->connect_timeout = u->conf->connect_timeout;
更改为:
map<domain, ower_conf> domain_conf;

get_upstream_connect_timeout(doamin, u)
{
    conf = domain_conf[domain]
    if conf && conf->upstream_connect_timeout {
        return conf->upstream_connect_timeout
    }

    return u->conf->connect_timeout;
}

u->connect_timeout = get_upstream_connect_timeout(domain, u)
```

```cassandraql
ngx指令变量化
参考 ngx_http_variables.c  limit_rate指令的实现
```
