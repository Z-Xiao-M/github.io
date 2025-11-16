#  ONNX 简单介绍
ONNX（Open Neural Network Exchange）由微软与Facebook于2017年联合推出，推出后迅速得到了各大厂商和框架的支持。
<img width="1254" height="806" alt="Image" src="https://github.com/user-attachments/assets/ee4e1b12-6c23-4b62-bc17-7a86832a52f3" />
其核心机制在于定义了一套与平台无关的统一算子和计算图规范：无论使用何种训练框架（如: PyTorch、TensorFlow、Scikit-learn、MXNet）构建模型，只需将其转换为ONNX格式，就能在任何支持ONNX Runtime的环境中运行——从云端服务器到边缘设备，从移动端到Web浏览器。这种标准化不仅实现了真正的"一次训练，随处部署"，更构建起一个开放的跨生态协作网络。让开发者不必疲于应对环境适配，得以挣脱框架锁定的束缚，从而专注算法创新。

想了解更多详细信息请关注[ONNX | Home](https://onnx.ai/)或[GitHub](https://github.com/onnx/onnx)

#  ONNX Runtime 简单介绍
ONNX Runtime（ORT）是微软推出的跨平台、高性能、专为ONNX 模型打造的AI **推理引擎**，其核心价值在于为 ONNX 格式模型提供统一、高效的运行环境：它支持跨平台部署，兼容 Windows、Linux、macOS、Android、iOS 等系统及 x86、ARM 等芯片架构，能适配 CPU、GPU、NPU 等各类硬件；同时提供 Python、C++、C#、JavaScript、Java 等多种API开发接口。

<img width="1505" height="595" alt="Image" src="https://github.com/user-attachments/assets/d98c729c-42e5-4965-a751-b80441132850" />

想了解更多详细信息请关注[ONNX Runtime | Home](https://onnxruntime.ai/)或[GitHub](https://github.com/microsoft/onnxruntime)

# 现实中的痛点
**ML模型开发者的痛点**
- 主要使用Python生态系统（Transformers, scikit-learn, pandas）
- 训练数据多为CSV文件格式
- 擅长训练但不擅长将模型部署为API服务

**后端开发者的痛点**
- 熟悉调用外部API和DBMS，习惯结构化响应（如JSON）
- 不了解如何执行ML模型
- 担心ML模型作为资源消耗大的黑盒带来的潜在风险
- 缺乏GPU资源管理经验

**引申出三个需求**
- 想要用数据库中的数据做推理，该怎么办？
- 想要把推理结果存入数据库中并用于查询过滤，该怎么办？
- 想要自动化这一切，该怎么办？

# pg_onnx 
pg_onnx 是 PostgreSQL 的扩展插件，核心是通过**集成 ONNX Runtime，让机器学习模型在数据库内直接完成推理**，无需跨系统迁移数据。
- 打通 “数据库存储” 与 “ML 推理” 的链路，解决数据迁移、跨系统协作的效率问题。
- 适配主流框架导出的 ONNX 模型，支持在 PostgreSQL 内部完成模型加载、会话管理和推理执行。
- 消除协作壁垒：适配 ML 模型师（Python 生态）和后端开发（DB/API 生态）的使用习惯。
- 提升推理效率：数据无需导出，减少传输延迟，支持 CUDA GPU 加速。
- 简化技术架构：无需额外搭建推理服务，直接通过 SQL 函数或触发器调用模型。
- 提供完整模型管理函数：支持 ONNX 模型的导入、删除、列表查询和详情查看。
- 支持会话管理：可创建、执行、销毁推理会话，灵活适配不同推理场景。
- 触发器联动：能通过 PostgreSQL 触发器，在数据插入 / 更新时自动触发推理并存储结果。

整体架构如下图所示，来自作者23年pgday的[pdf](https://pgday.postgresql.kr/static/pgday-2023pg_onnx.pdf)分享

<img width="1145" height="511" alt="Image" src="https://github.com/user-attachments/assets/d9408f4d-a9ba-4fdd-b8bb-1fc84b0c52a7" />

采用 “PostgreSQL 扩展 + ONNX Runtime+Background Worker” 分离设计

- pg_onnx作为PostgreSQL 扩展，负责 SQL 入口接收请求（其实就是pg_onnx提供了一系列的函数）
```sql
postgres=# create extension pg_onnx;
CREATE EXTENSION
postgres=# select proname from pg_proc where proname like 'pg_onnx%';
          proname          
---------------------------
 pg_onnx_inspect_model_bin
 pg_onnx_list_model
 pg_onnx_import_model
 pg_onnx_drop_model
 pg_onnx_list_session
 pg_onnx_create_session
 pg_onnx_describe_session
 pg_onnx_destroy_session
 pg_onnx_execute_session
```
- onnxruntime-server其实就是封装了ONNX Runtime，作为Background Worker，提供服务。比如说：集中执行推理，会话管理以及CUDA GPU 加速。

```bash
postgres=# \! ps -aux|grep postgres:
postgres  160845  0.0  0.0  70648  4568 ?        Ss   22:18   0:00 postgres: logger 
postgres  160846  0.0  0.0 217472 10016 ?        Ss   22:18   0:00 postgres: checkpointer 
postgres  160847  0.0  0.0 217492  6812 ?        Ss   22:18   0:00 postgres: background writer 
postgres  160849  0.0  0.0 217340  9592 ?        Ss   22:18   0:00 postgres: walwriter 
postgres  160850  0.0  0.0 218944  8796 ?        Ss   22:18   0:00 postgres: autovacuum launcher 
postgres  160851  0.0  0.0 218920  7964 ?        Ss   22:18   0:00 postgres: logical replication launcher 
postgres  160853  0.0  0.1 243376 28624 ?        Ss   22:18   0:00 postgres: postgres postgres [local] idle
postgres  160854  0.0  0.0 346744 12924 ?        Ssl  22:18   0:00 postgres: pg_onnx 
```

pg_onnx和onnxruntime-server，其实就是标准的CS模型，一个作为客户端，一个作为服务端。

二者通过 TCP/IP 通信避免多进程重复加载模型导致的资源枯竭。对于通信方面更具体一点，更具体一点其实是本地回环，否者网络消耗又上去了。

另外值得注意的是，在导入模型时，会将传入的模型作为大对象来管理。

# 生成线性回归模型
线性回归是机器学习中最简单的模型，前不久薛晓刚薛老师也写了一篇关于相似的文章[Oracle和MySQL数据库中做线性回归](https://mp.weixin.qq.com/s/nBQKPBZyOgRq8eWvtxuj0w)，感兴趣的朋友可以再去了解了解，写的比我详细。**回到正文**，这里我没有使用onnx自身定义的算子，去生成线性回归模型，而是使用的**pytorch**。你可以参考官方文档的[ONNX with Python](https://onnx.ai/onnx/intro/python.html#a-simple-example-a-linear-regression)写法来实现这个线性回归模型。
```python
import numpy as np
import torch
from torch import nn

class UnaryLinearReg(nn.Module):
    def __init__(self):
        super().__init__()
        self.linear = nn.Linear(1, 1)

    def forward(self, x):
        return self.linear(x)

def main():
    # y ≈ 3x + 2 + noise
    np.random.seed(0)
    x = np.random.uniform(0, 10, size=(10000, 1)).astype(np.float32)
    noise = np.random.normal(0, 0.5, size=(10000, 1)).astype(np.float32)
    labels = 3 * x + 2 + noise

    model = UnaryLinearReg()
    loss_fn = nn.MSELoss()
    optimizer = torch.optim.SGD(model.parameters(), lr=0.01)

    epochs = 20
    batch_size = 100
    for epoch in range(epochs):
        for i in range(0, len(x), batch_size):
            batch_x = torch.from_numpy(x[i:i + batch_size])
            batch_y = torch.from_numpy(labels[i:i + batch_size])

            optimizer.zero_grad()
            pred = model(batch_x)
            loss = loss_fn(pred, batch_y)
            loss.backward()
            optimizer.step()

        if epoch % 2 == 0:
            print(f"epoch {epoch:2d} | loss {loss.item():.4f}")

    w, b = model.parameters()
    print(f"w = {w.item():.3f}, b = {b.item():.3f}")

    torch.onnx.export(
        model,
        torch.randn(1, 1), 
        "model.onnx",
        input_names=['x'], 
        output_names=['y'], 
        dynamic_shapes={'x': {0: 'batch'}},
        opset_version=18,
    )

if __name__ == '__main__':
    main()
```

# 其他注意事项
### PostgreSQL的编译选项需要附上`--enable-nls`
否则在编译pg_onnx时，可能会出现下面的编译错误提示，没有花时间仔细研究，到底是为什么
```bash
In file included from /u01/app/halo/product/dbms/16/include/postgresql/server/postgres.h:45,
                 from /home/postgres/code/16/contrib/pg_onnx/pg_onnx/bridge/bgworker_side/../../pg_onnx.hpp:14,
                 from /home/postgres/code/16/contrib/pg_onnx/pg_onnx/bridge/bgworker_side/bgworker_side.hpp:8,
                 from /home/postgres/code/16/contrib/pg_onnx/pg_onnx/bridge/bgworker_side/bgworker.cpp:5:
/usr/include/libintl.h:39:14: error: expected unqualified-id before ‘const’
   39 | extern char *gettext (const char *__msgid)
      |              ^~~~~~~
/usr/include/libintl.h:39:14: error: expected ‘)’ before ‘const’
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1190:20: note: to match this ‘(’
 1190 | #define gettext(x) (x)
      |                    ^
/usr/include/libintl.h:44:14: error: expected unqualified-id before ‘const’
   44 | extern char *dgettext (const char *__domainname, const char *__msgid)
      |              ^~~~~~~~
/usr/include/libintl.h:44:14: error: expected ‘)’ before ‘const’
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1191:23: note: to match this ‘(’
 1191 | #define dgettext(d,x) (x)
      |                       ^
/usr/include/libintl.h:61:14: error: expected unqualified-id before ‘unsigned’
   61 | extern char *ngettext (const char *__msgid1, const char *__msgid2,
      |              ^~~~~~~~
/usr/include/libintl.h:61:14: error: expected ‘)’ before ‘unsigned’
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1192:26: note: to match this ‘(’
 1192 | #define ngettext(s,p,n) ((n) == 1 ? (s) : (p))
      |                          ^
/usr/include/libintl.h:61:14: error: expected ‘)’ before ‘unsigned’
   61 | extern char *ngettext (const char *__msgid1, const char *__msgid2,
      |              ^~~~~~~~
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1192:25: note: to match this ‘(’
 1192 | #define ngettext(s,p,n) ((n) == 1 ? (s) : (p))
      |                         ^
/usr/include/libintl.h:67:14: error: expected unqualified-id before ‘unsigned’
   67 | extern char *dngettext (const char *__domainname, const char *__msgid1,
      |              ^~~~~~~~~
```
 
### max supported IR version: 11 和 opset 23
```sql
test=# SELECT pg_onnx_import_model(
               'simple_model',
               'v2', 
               PG_READ_BINARY_FILE('/home/postgres/model/simple_model.onnx')::bytea,
               '{"cuda": false}'::jsonb, 
               'simple_model'
       );
ERROR:  pg_onnx_inspect_model_bin: /onnxruntime_src/onnxruntime/core/graph/model.cc:181 onnxruntime::Model::Model(onnx::ModelProto&&, const onnxruntime::PathString&, const onnxruntime::IOnnxRuntimeOpSchemaRegistryList*, const onnxruntime::logging::Logger&, const onnxruntime::ModelOptions&) Unsupported model IR version: 12, max supported IR version: 11
test=# SELECT pg_onnx_import_model(
               'simple_model',
               'v2', 
               PG_READ_BINARY_FILE('/home/postgres/model/simple_model.onnx')::bytea,
               '{"cuda": false}'::jsonb, 
               'simple_model'
       );
ERROR:  pg_onnx_inspect_model_bin: /onnxruntime_src/onnxruntime/core/graph/model_load_utils.h:46 void onnxruntime::model_load_utils::ValidateOpsetForDomain(const std::unordered_map<std::__cxx11::basic_string<char>, int>&, const onnxruntime::logging::Logger&, bool, const std::string&, int) ONNX Runtime only *guarantees* support for models stamped with official released onnx opset versions. Opset 24 is under development and support for this is limited. The operator schemas and or other functionality may change before next ONNX release and in this case ONNX Runtime will not guarantee backward compatibility. Current official support for domain ai.onnx is till opset 23.
```
### 输入张量的形状计算不对
这里的det的原始输入张量形状为(-1,3,-1,-1)，实际输入的coordinates.json是一个形状为(1, 3, 160, 160)，总元素数量76800，报错如下
```sql
(onnx-env) postgres@zxm-VMware-Virtual-Platform:~$ psql test
psql (16.10)
Type "help" for help.

test=# select * from pg_onnx_list_model() where name = 'det';
 name | version |     option      |            inputs            |                 outputs                 | description |          created_at           | lo_oid 
------+---------+-----------------+------------------------------+-----------------------------------------+-------------+-------------------------------+--------
 det  | v1      | {"cuda": false} | {"x": "float32[-1,3,-1,-1]"} | {"fetch_name_0": "float32[-1,1,-1,-1]"} | det         | 2025-10-22 17:59:12.502754+08 |  33180
(1 row)

test=# SELECT pg_onnx_execute_session(
    'det', 
    'v1', 
    pg_read_file('/home/postgres/OnnxOCR/coordinates.json')::jsonb
);
ERROR:  pg_onnx_internal_execute_session: tried creating tensor with negative value in shape
```
原因在于，pg_onnx依赖的子项目[onnxruntime-server](https://github.com/kibae/onnxruntime-server)对于多维张量的计算是存在问题的，在计算的过程中会调用input_value::batched_shape这个函数，函数实现如下
```c++
std::vector<int64_t>
onnxruntime_server::onnx::execution::input_value::batched_shape(const std::vector<int64_t> &shape, size_t value_count) {
	// check shape contains -1
	if (std::find(shape.begin(), shape.end(), -1) == shape.end())
		return shape;

	// calculate batch size
	std::vector<int64_t> shape_copy = shape;
	int64_t batch = (int64_t)value_count;
	for (auto &s : shape_copy) {
		if (s != -1)
			batch = (int64_t)std::max((double)1, std::ceil((double)batch / (double)s));
	}

	// replace -1 with batch size
	for (auto &s : shape_copy) {
		if (s == -1) {
			s = batch;
			break;
		}
	}

	return shape_copy;
}
```
而输入这是原始张量形状和当前输入张量的元素总数分别是(-1, 3, -1, -1)和76800，导致最终计算结果为(25600, 3, -1, -1)，这和实际输入张量的形状(1, 3, 160, 160)完全是对不上的，所以导致报错。然后我就尝试着修复了一下, **值得注意的是，对于bool类型的张量还是原来的处理逻辑，因为我没有这个测试场景的需求，所以待测试**
<img width="925" height="767" alt="Image" src="https://github.com/user-attachments/assets/7669678e-d792-4e77-9b07-64546b8d1e54" />