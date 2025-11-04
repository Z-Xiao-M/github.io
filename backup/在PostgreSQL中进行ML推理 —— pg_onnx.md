## [onnx](https://github.com/onnx/onnx)
ONNX （Open Neural Network Exchange）是一种用于表示机器学习模型的开放格式。 ONNX 定义了一组通用的运算符 - 机器学习和深度学习模型的构建块 - 以及一个通用的文件格式，使 AI 开发人员能够使用各种框架、工具、运行时和编译器来使用模型。

## onnxruntime

## pg_onnx

## 一些限制和BUG修复
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
而输入这是原始张量形状和当前输入张量的元素总数分别是(-1, 3, -1, -1)和76800，导致最终计算结果为(25600, 3, -1, -1)，这和实际输入张量的形状(1, 3, 160, 160)完全是对不上的，所以导致报错。然后我就尝试着修复了一下

<img width="925" height="767" alt="Image" src="https://github.com/user-attachments/assets/7669678e-d792-4e77-9b07-64546b8d1e54" />