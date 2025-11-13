#  ONNX 简单介绍
ONNX（Open Neural Network Exchange）是人工智能领域的开放标准文件格式，由微软与Facebook于2017年联合推出，推出后迅速得到了各大厂商和框架的支持

<img width="1254" height="806" alt="Image" src="https://github.com/user-attachments/assets/ee4e1b12-6c23-4b62-bc17-7a86832a52f3" />

该标准通过定义一组与平台、环境无关的统一算子集和文件格式，为AI模型互操作性提供了通用协议。其工作机制简洁而强大：开发者可自由选择PyTorch、TensorFlow、Scikit-learn、MXNet等主流框架训练模型，再将其转换为标准化的ONNX模型。这种统一表达使模型能够轻松部署于任何兼容ONNX Runtime的运行环境中，真正实现"一次训练，随处运行"的愿景。

#  ONNX Runtime 简单介绍
ONNX Runtime 是微软推出的高性能推理引擎，专为运行 ONNX 模型设计，更是实现 AI 模型跨平台部署的核心执行器。它具备强大的跨平台支持能力，兼容 Windows、Linux、macOS，以及 IoT 边缘设备、移动端、Web 浏览器等多种环境；硬件加速方面内置 CPU、GPU（含 CUDA/TensorRT/DirectML 等主流后端）、NPU（适配 Intel、高通等厂商）等多种加速后端，可自动适配最优硬件；性能上通过算子融合、内存优化等技术，在多数场景下推理速度比原训练框架快 2-5 倍；同时提供 C++、Python、C#、Java、JavaScript 等多语言 API，方便集成到各类应用中，且已被微软、亚马逊、Adobe、Meta 等企业广泛用于大规模线上服务，具备成熟的生产级可用性。

# 现实痛点
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
pg_onnx给出的答案是——在PostgreSQL中直接集成ONNX推理能力。


整体架构如下图所示，来自作者23年pgday的[pdf](https://pgday.postgresql.kr/static/pgday-2023pg_onnx.pdf)分享

<img width="1145" height="511" alt="Image" src="https://github.com/user-attachments/assets/d9408f4d-a9ba-4fdd-b8bb-1fc84b0c52a7" />


# 其他注意事项
### PostgreSQL的编译选项需要附上`--enable-nls`
否则可能会出现下面的编译错误提示，没有花时间仔细研究，到底是为什么
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
而输入这是原始张量形状和当前输入张量的元素总数分别是(-1, 3, -1, -1)和76800，导致最终计算结果为(25600, 3, -1, -1)，这和实际输入张量的形状(1, 3, 160, 160)完全是对不上的，所以导致报错。然后我就尝试着修复了一下, **对于bool类型的张量还是原来的处理逻辑，待测试**
<img width="925" height="767" alt="Image" src="https://github.com/user-attachments/assets/7669678e-d792-4e77-9b07-64546b8d1e54" />