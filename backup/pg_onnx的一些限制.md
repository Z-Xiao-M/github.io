max supported IR version: 11 和 opset 23
```sql
test=# SELECT pg_onnx_import_model(
               'simple_model',
               'v2', 
               PG_READ_BINARY_FILE('/home/postgres/model/simple_model.onnx')::bytea,
               '{"cuda": false}'::jsonb, 
               'simple_model'
       );
错误:  pg_onnx_inspect_model_bin: /onnxruntime_src/onnxruntime/core/graph/model.cc:181 onnxruntime::Model::Model(onnx::ModelProto&&, const onnxruntime::PathString&, const onnxruntime::IOnnxRuntimeOpSchemaRegistryList*, const onnxruntime::logging::Logger&, const onnxruntime::ModelOptions&) Unsupported model IR version: 12, max supported IR version: 11

背景:  在赋值的第7行的PL/pgSQL函数pg_onnx_import_model(text,text,bytea,jsonb,text)
test=# SELECT pg_onnx_import_model(
               'simple_model',
               'v2', 
               PG_READ_BINARY_FILE('/home/postgres/model/simple_model.onnx')::bytea,
               '{"cuda": false}'::jsonb, 
               'simple_model'
       );
错误:  pg_onnx_inspect_model_bin: /onnxruntime_src/onnxruntime/core/graph/model_load_utils.h:46 void onnxruntime::model_load_utils::ValidateOpsetForDomain(const std::unordered_map<std::__cxx11::basic_string<char>, int>&, const onnxruntime::logging::Logger&, bool, const std::string&, int) ONNX Runtime only *guarantees* support for models stamped with official released onnx opset versions. Opset 24 is under development and support for this is limited. The operator schemas and or other functionality may change before next ONNX release and in this case ONNX Runtime will not guarantee backward compatibility. Current official support for domain ai.onnx is till opset 23.

背景:  在赋值的第7行的PL/pgSQL函数pg_onnx_import_model(text,text,bytea,jsonb,text)
```