## DBpack 分布式事务中间件赋能 python 微服务

### 什么是分布式事务

事务处理几乎在每一个信息系统中都会涉及，它存在的意义是为了保证系统数据符合期望的，且相互关联的数据之间不会产生矛盾，即数据状态的一致性。

按照数据库的经典理论，原子性、隔离性、持久性。原子性要求数据要么修改要么回滚，隔离性要求事务之间相互独立不影响，持久性要求事务的执行能正确的持久化，不丢失数据。mysql 类的行式数据库通过 mvcc 多版本视图和 wal 预写日志等技术的协作，实现了单个服务使用单个数据源或者单个服务使用多个数据源场景的多事务的原子性、隔离性和持久性。

凤凰架构这本书中有描述，单个服务使用单个数据源称之为本地事务，单个服务使用多个数据源称之为全局事务，而分布式事务特指多个服务同时访问多个数据源的事务处理机制。

### DBpack 简介

分布式事务的实现有很多方式，如可靠性事务队列，TCC事务，SAGA事务等。

可靠性事务队列，也就是BASE，听起来和强一致性的ACID，"酸碱"格格不入，它作为最终一致性的概念起源，系统性地总结了一种针对分布式事务的技术手段。

TCC 较为烦琐，如同名字所示，它分为以下三个阶段。

- **Try**：尝试执行阶段，完成所有业务可执行性的检查（保障一致性），并且预留好全部需用到的业务资源（保障隔离性）。
- **Confirm**：确认执行阶段，不进行任何业务检查，直接使用 Try 阶段准备的资源来完成业务处理。Confirm 阶段可能会重复执行，因此本阶段所执行的操作需要具备幂等性。
- **Cancel**：取消执行阶段，释放 Try 阶段预留的业务资源。Cancel 阶段可能会重复执行，也需要满足幂等性。

SAGA 事务将事务进行了拆分，大事务拆分若干个小事务，将整个分布式事务 T 分解为 n 个子事务，同时为为每一个子事务设计对应的补偿动作。尽管补偿操作通常比冻结或撤销容易实现，但保证正向、反向恢复过程的能严谨地进行也需要花费不少的工夫。

DBPack 的分布式事务致力于实现对用户的业务无入侵，使用时下流行的sidecar架构，主要使用 ETCD Watch 机制来驱动分布式事务提交回滚，它对 HTTP 流量和 MYSQL 流量做了拦截代理，支持 AT 模式（自动补偿 SQL）和 TCC 模式（自动补偿 HTTP 请求）。

DBPack 的 AT 模式性能取决于全局锁的释放速度，哪个事务竞争到了全局锁就能对业务数据做修改，在单位时间内，全局锁的释放速度越快，竞争到锁的事务越多，性能越高。DBPack 创建全局事务、注册分支事务只是在 ETCD 插入两条 KV 数据，事务提交回滚时修改对应数据的状态，通过 ETCD Watch 机制感知到数据的变化就能立即处理数据的提交回滚，从而在交互上减少了很多 RPC 请求。

![distributed transaction-sidecar.drawio](../images/distributed-transaction-sidecar.drawio.png)

DBPack 的 TCC 模式中，请求会先到达 sidecar 后再注册 TCC 事务分支，确保 Prepare 先于 Cancel 执行。具体到操作的业务数据，建议使用 XID 和 BranchID 加锁。

![tcc.drawio](../images/tcc.drawio.png)

### DBpack 赋能 python 微服务

以上的前戏已铺垫，后文以讲解python 微服务代码为主，不涉及 dbpack 源码，感兴趣的童鞋可去自行调试了解。

> https://github.com/CECTC/dbpack-samples/blob/main/python

这里会提到三个微服务，首先是是事务发起方，其次是订单系统，最后是产品库存系统。而每一个微服务，都使用dbpack代理。事务发起方请求成功后，当订单正常commit后，产品库存要发生正常扣除，一旦一个微服务未完成，另一个则要发生回滚，也就是说，两个微服务系统要保持一致。

首先，模拟分布式事务发起方的服务，该服务会注册两个 handler，一个会发起正常的请求，走 dbpack 代理发起分布式事务，另一个会则会非正常返回。事务发起方会根据 http 的请求情况，决定是否要发起分布式事务回滚。

以下借用了 flask web 框架实现了事务发起方的两个handler，通过两个http请求我们可以模拟分布式事务发起或者回滚。 

```
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

create_so_url        = "http://order-svc:3001/createSo"
update_inventory_url = "http://product-svc:3002/allocateInventory"

@app.route('/v1/order/create', methods=['POST'])
def create_1():
   return create_so(rollback=False)
    
@app.route('/v1/order/create2', methods=['POST'])
def create_2():
   return create_so(rollback=True)

def create_so(rollback=True):
    xid = request.headers.get("x-dbpack-xid")

    so_items = [dict(
        product_sysno=1,
        product_name="apple iphone 13",
        original_price=6799,
        cost_price=6799,
        deal_price=6799,
        quantity=2,
    )]

    so_master = [dict(
        buyer_user_sysno = 10001,
        seller_company_code = "SC001",
        receive_division_sysno = 110105,
        receive_address = "beijing",
        receive_zip = "000001",
        receive_contact = "scott",
        receive_contact_phone =  "18728828296",
        stock_sysno = 1,
        payment_type = 1,
        so_amt = 6999 * 2,
        status = 10,
        appid = "dk-order",
        so_items = so_items,
    )]

    success = (jsonify(dict(success=True, message="success")), 200)
    failed = (jsonify(dict(success=False, message="failed")), 400)
    headers = {
        "Content-Type": "application/json",
        "xid": xid
    }

    so_req = dict(req=so_master)
    resp1 = requests.post(create_so_url, headers=headers, json=so_req)
    if resp1.status_code == 400:
        return failed

    ivt_req = dict(req=[dict(product_sysno= 1, qty=2)])
    resp2 = requests.post(update_inventory_url, headers=headers, json=ivt_req)
    if resp2.status_code == 400:
        return failed

    if rollback:
        print("rollback")
        return failed

    return success

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=3000)
```

那么如何使用 dbpack 代理该服务呢？

```
$./dist/dbpack start --config ../dbpack-samples/configs/config-aggregation.yaml

$ cat ../dbpack-samples/configs/config-aggregation.yaml
listeners:
  - protocol_type: http
    socket_address:
      address: 0.0.0.0
      port: 13000
    config:
      backend_host: aggregation-svc:3000
    filters:
      - httpDTFilter

filters:
  - name: httpDTFilter
    kind: HttpDistributedTransaction
    conf:
      appid: aggregationSvc
      transaction_infos:
        - request_path: "/v1/order/create"
          timeout: 60000
        - request_path: "/v1/order/create2"
          timeout: 60000

distributed_transaction:
  appid: aggregationSvc
  retry_dead_threshold: 130000
  rollback_retry_timeout_unlock_enable: true
  etcd_config:
    endpoints:
      - etcd:2379
```

可想而知，以上的微服务两个 handler 是通过 filters这部分的定义来配置拦截的。

接着是订单系统。

```
from flask import Flask, jsonify, request
from datetime import datetime
import mysql.connector

import time
import random

app = Flask(__name__)

insert_so_master = "INSERT /*+ XID('{xid}') */ INTO order.so_master({keys}) VALUES ({placeholders})"
insert_so_item = "INSERT /*+ XID('{xid}') */ INTO order.so_item({keys}) VALUES ({placeholders})"

def conn():
    retry = 0
    while retry < 3:
        time.sleep(5)
        try:
            c = mysql.connector.connect(
              host="dbpack3",
              port=13308,
              user="dksl",
              password="123456",
              database="order",
              autocommit=True,
            )
            if c.is_connected():
                db_Info = c.get_server_info()
                print("Connected to MySQL Server version ", db_Info)
                return c
        except Exception as e:
            print(e.args)
        retry += 1 
 
connection = conn()
cursor = connection.cursor(prepared=True,)

@app.route('/createSo', methods=['POST'])
def create_so():
    xid = request.headers.get('xid')
    reqs = request.get_json()
    if xid and "req" in reqs:
        for res in reqs["req"]:
            res["sysno"] = next_id()
            res["so_id"] = res["sysno"]
            res["order_date"] = datetime.now()
            res_keys = [str(k) for k,v in res.items() if k != "so_items" and str(v) != ""]
            so_master = insert_so_master.format(
                xid=xid,
                keys=", ".join(res_keys),
                placeholders=", ".join(["%s"] * len(res_keys)),
            )

            try:
                cursor.execute(so_master, tuple(res.get(k, "") for k in res_keys))
            except Exception as e:
                print(e.args)
             
            so_items = res["so_items"]
            for item in so_items:
                item["sysno"] = next_id()
                item["so_sysno"] = res["sysno"]
                item_keys = [str(k) for k,v in item.items() if str(v) != "" ]
                so_item = insert_so_item.format(
                    xid=xid,
                    keys=", ".join(item_keys),
                    placeholders=", ".join(["%s"] * len(item_keys)),
                )
                try:
                    cursor.execute(so_item, tuple(item.get(k, "") for k in item_keys))
                except Exception as e:
                    print(e.args)
 
        return jsonify(dict(success=True, message="success")), 200
    
    return jsonify(dict(success=False, message="failed")), 400 

def next_id():
    return random.randrange(0, 9223372036854775807)
  

if __name__ == '__main__':
   app.run(host="0.0.0.0", port=3001)
```

注意到 sql 中以注解的形式添加使用了 xid ，主要是方便配合 dbpack 识别后做出相应的分布式事务处理，也就是回滚还是commit。

这里数据库连接使用 autocommit 这种方式。同时，使用 python 中的 mysql.connector 这个 lib 来支持 sql 传输中的二段式加密传输协议，见代码中声明的`prepared=true`。

用以下命令，使用 dbpack 代理 order 微服务：

```
./dist/dbpack start --config ../dbpack-samples/configs/config-order.yaml
```

最后是产品库存系统，详细代码如下：

```
from flask import Flask, jsonify, request

import time
import mysql.connector

app = Flask(__name__)

allocate_inventory_sql = "update /*+ XID('{xid}') */ product.inventory set available_qty = available_qty - %s, allocated_qty = allocated_qty + %s where product_sysno = %s and available_qty >= %s;"

def conn():
    retry = 0
    while retry < 3:
        time.sleep(5)
        try:
            c = mysql.connector.connect(
              host="dbpack2",
              port=13307,
              user="dksl",
              password="123456",
              database="product",
              autocommit=True,
            )

            if c.is_connected():
                db_Info = c.get_server_info()
                print("Connected to MySQL Server version ", db_Info)
                return c
        except Exception as e:
            print(e.args)
        retry += 1 
 
connection = conn()
cursor = connection.cursor(prepared=True,)

@app.route('/allocateInventory', methods=['POST'])
def create_so():
    xid = request.headers.get('xid')
    reqs = request.get_json()
    if xid and "req" in reqs:
        for res in reqs["req"]:
            try:
                cursor.execute(allocate_inventory_sql.format(xid=xid), (res["qty"], res["qty"], res["product_sysno"], res["qty"],))
            except Exception as e:
                print(e.args)
        
        return jsonify(dict(success=True, message="success")), 200
        
    return jsonify(dict(success=False, message="failed")), 400

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=3002)
```

同样，用以下命令使用 dbpack 代理 product 微服务：

```
./dist/dbpack start --config ../dbpack-samples/configs/config-product.yaml
```

我们可以使用docker-compose一键拉起以上三个微服务：

```
docker-compose up
```

正常情况下，以下请求会触发订单系统和产品库存系统的正常 commit：

```
curl -XPOST http://localhost:13000/v1/order/create
```

而以下命令虽然正常请求了订单系统和产品库存的 API，不管事务是否正常执行，由于事务发起方状态码不正常，要求"回滚"，所以会导致已经 commit 的微服务发生回滚，以此保证分布式系统的一致性：

```
curl -XPOST http://localhost:13000/v1/order/create2
```

#### 参考资料

- 官方仓库：https://github.com/CECTC/dbpack； https://github.com/CECTC/dbpack-samples
- 凤凰架构：http://icyfenix.cn/architect-perspective/general-architecture/transaction/distributed.html
- DBpack 博客： https://www.cnblogs.com/DKSL/p/dbpack.html
