> [!CAUTION]
> 以下内容来自AI总结 如有错误 需自行甄别 
> 此处用作记录 参考 https://berthub.eu/articles/posts/practical-peg-parsing/ 生成
> 或许可能替代Lex/Yacc 完成SQL的词法语法解析 duckdb有望成为先行者 官方文章[Runtime-Extensible SQL Parsers Using PEG](https://duckdb.org/2024/11/22/runtime-extensible-parsers.html)

在软件开发中，“解析”是个绕不开的需求——小到解析配置文件里的向量数据，大到处理Prometheus监控指标、JSON/XML格式，甚至自定义协议。过去，开发者要么靠正则表达式硬啃，要么用Lex/Yacc这类传统工具搭建解析器，但前者易出错、难维护，后者配置复杂、与现代语言集成度低。直到**Parsing Expression Grammars（PEG，解析表达式语法）** 与轻量级库**cpp-peglib**的出现，才让“写一个健壮的解析器”从“麻烦事”变成了“随手能做的事”。


## 一、PEG：比传统语法更“懂开发者”的解析逻辑
PEG之所以能替代Lex/Yacc，核心在于它抛弃了传统LALR语法的“歧义处理”“冲突解决”等复杂概念，用更贴近人类思维的规则定义解析逻辑。它的关键特性的是“**有序选择**”和“**前瞻断言**”，这两个能力直接解决了很多解析场景的痛点。

### 1.  PEG的核心规则与运算符
与传统语法不同，PEG的规则更像“一步一步描述如何匹配文本”，核心运算符只有几个，但足够灵活：
- `Rule <- Expression`：定义规则，比如`Vector <- '(' Number (',' Number)* ')'`表示“Vector是左括号+一个Number+零个或多个‘逗号+Number’+右括号”；
- `/`：**有序选择**，不是“或”（OR），而是“按顺序尝试，第一个匹配成功的生效”。比如`Char <- ('\\.' ) / (!'\\' .)`，会先尝试匹配转义字符（如`\"`），失败了再匹配普通字符；
- `!`：**否定前瞻断言**，“看一眼下一个字符，如果是目标字符就不匹配当前规则”，且不消费文本。比如解析字符串时用`!'"' Char`，确保遇到双引号时停止，避免提前终止字符串；
- `*`/`+`：分别表示“零或多次重复”“一次或多次重复”，与正则类似，但更严格（不会出现正则的“贪婪/非贪婪”混乱）；
- `%whitespace <- [\t ]*`：cpp-peglib的语法糖，自动忽略规则间的空格、制表符，不用手动在每个规则里加空格匹配。


### 2.  PEG vs Lex/Yacc：为什么选PEG？
传统的Lex/Yacc是“词法分析（Lex）+语法分析（Yacc）”分离的模式，需要分别写词法规则和语法规则，还得处理“移进-归约冲突”等问题，对新手极不友好。而PEG的优势很明确：
- 无歧义：有序选择确保“只有一个正确匹配路径”，不用处理冲突；
- 自顶向下解析：直接从“最终目标”（如Vector、Prometheus指标行）拆解到“最小单元”（如Number、字符），逻辑更直观；
- 无需分离词法与语法：一个规则就能覆盖“词法+语法”，比如`Number <- [+-]?[0-9]*([.][0-9]*)?`既定义了数字的格式，也完成了词法匹配。


## 二、cpp-peglib：让PEG在C++里“开箱即用”
cpp-peglib是一个单文件C++库（只需包含`peglib.h`），无需预处理、无外部依赖，甚至能直接解析docx这类复杂格式。它的核心设计理念是“简单的事简单做，复杂的事能做到”——用几行代码就能实现基础解析，扩展后也能处理HTML、XML等复杂场景。

### 实战1：解析向量数据（入门案例）
比如要解析`(0.123, -.987, 2.4, 3)`这样的向量，用cpp-peglib只需3步：定义语法、绑定语义动作、执行解析。

#### 1. 定义PEG语法
首先用PEG描述向量的格式，明确“Vector由什么组成”“Number是什么样的”：
```cpp
#include "peglib.h"
#include <vector>
#include <fmt/format.h> // 用于打印结果

int main(int argc, char** argv) {
    // 1. 定义PEG语法
    peg::parser parser(R"(
        Vector      <- '(' Number (',' Number)* ')'  # 向量结构：(数字, 数字, ...)
        Number      <- [+-]?[0-9]*([.][0-9]*)?       # 数字：支持正负、整数、小数
        %whitespace <- [\t ]*                        # 自动忽略空格、制表符
    )");
```

#### 2. 绑定“语义动作”：把匹配的文本转成数据
语法只负责“匹配文本”，而我们需要的是“解析出的数值”（比如把`"0.123"`转成`double`）。cpp-peglib允许给每个规则绑定一个函数，这个函数就是“语义动作”：
```cpp
    // 给Number规则绑定动作：将匹配的文本转成double
    parser["Number"] = [](const peg::SemanticValues& vs) {
        // vs.token_to_number<double>()：直接把匹配的文本转成double
        return vs.token_to_number<double>();
    };

    // 给Vector规则绑定动作：收集所有Number的结果，组成vector<double>
    parser["Vector"] = [](const peg::SemanticValues& vs) {
        // vs是vector<std::any>，存储每个子规则（这里是Number）的返回值
        std::vector<double> result;
        for (const auto& val : vs) {
            // 把std::any转成double，存入result
            result.push_back(std::any_cast<double>(val));
        }
        // 也可以用cpp-peglib的简写：return vs.transform<double>();
        return result;
    };
```

#### 3. 执行解析并输出结果
最后调用`parser.parse()`，传入要解析的字符串，就能得到解析后的向量：
```cpp
    // 3. 执行解析
    std::vector<double> vector_result;
    if (parser.parse(argv[1], vector_result)) {
        // 用fmt打印结果
        fmt::print("解析结果：");
        for (size_t i = 0; i < vector_result.size(); ++i) {
            fmt::print("{}{}", vector_result[i], i == vector_result.size()-1 ? "\n" : ", ");
        }
    } else {
        fmt::print("解析失败！\n");
    }

    return 0;
}
```

运行程序测试，输入`./vector_parser "(0.123, -.987, 2.4, 3)"`，输出如下：
```
解析结果：0.123, -0.987, 2.4, 3
```

这个案例只用了20多行代码，比手动切割字符串、写正则验证的代码更简洁，且容错性更强（比如输入`( 1.2 , 3.4 )`带空格也能解析）。


## 三、攻克解析“老大难”：字符串转义处理
很多解析场景的痛点不是“匹配格式”，而是“处理转义”——比如解析`"This is a \"test\" string"`时，如何区分“转义的`"`”和“字符串结束的`"`”？PEG的“前瞻断言”和cpp-peglib的语义动作能轻松解决这个问题。

### 1. 定义支持转义的字符串语法
首先用PEG描述“带转义的字符串”：
```cpp
peg::parser str_parser(R"(
    QuotedString  <- "\"" String "\""  # 字符串："开头 + String + "结尾
    String        <- (!"\"" Char)*     # String：零或多个Char，直到遇到"
    Char          <- ('\\.' ) / (!'\\' .)  # Char：要么是转义字符（如\"），要么是普通字符
)");
```

这里的关键是`! "\"" Char`：用`! "\""`判断“下一个字符是不是`"`”，如果是就停止匹配String，确保`"`只作为字符串结束符；而`Char`的规则用`/`有序选择，优先匹配转义字符（避免把`\`当成普通字符）。

### 2. 处理转义：从“匹配文本”到“干净字符”
匹配到`\"`后，我们需要的是`"`而不是`\"`，所以要在语义动作里处理：
```cpp
// 给Char绑定动作：处理转义
str_parser["Char"] = [](const peg::SemanticValues& vs) {
    std::string matched = vs.token_to_string(); // 拿到匹配的文本（如\"或a）
    if (vs.choice() == 0) { // choice()返回“选择运算符/的第几个分支匹配成功”，0表示转义分支
        return matched.substr(1); // 截取\后面的字符（如\"→"）
    }
    return matched; // 普通字符直接返回
};

// 给String绑定动作：拼接所有Char的结果，组成完整字符串
str_parser["QuotedString"] = [](const peg::SemanticValues& vs) {
    std::string result;
    for (const auto& val : vs) {
        result += std::any_cast<std::string>(val);
    }
    return result;
};
```

测试输入`./string_parser "\"This is a \\\"test\\\" string\""`，输出如下：
```
解析结果：This is a "test" string
```

cpp-peglib还提供了一个简化技巧：用`< >`指定“需要保留的token部分”，比如把`Char`规则改成`Char <- ('\\' <.> ) / (!'\\' .)`，`<.>`表示“只保留`.`匹配的字符”（即`\`后面的部分），这样语义动作里就不用手动截取了：
```cpp
str_parser["Char"] = [](const peg::SemanticValues& vs) {
    return vs.token_to_string(); // 直接返回保留的部分，无需截取
};
```


## 四、实战升级：解析Prometheus监控指标
Prometheus指标格式是典型的“非 trivial 解析场景”——包含注释（HELP/TYPE/普通）、指标行（带可选标签和时间戳）、UTF-8支持和转义规则。用cpp-peglib，200行代码就能实现一个合规的解析器。

### 1. Prometheus指标格式回顾
一个标准的Prometheus指标文件如下：
```
# 普通注释
# HELP apt_autoremove_pending Apt packages pending autoremoval.
# TYPE apt_autoremove_pending gauge
apt_autoremove_pending 73
# HELP apt_upgrades_held Apt packages pending updates but held back.
# TYPE apt_upgrades_held gauge
apt_upgrades_held{arch="",origin=""} 0 1712003163000
```
核心结构包括：
- 注释行：`# HELP 指标名 描述`、`# TYPE 指标名 类型`、普通`# 注释`；
- 指标行：`指标名 [标签集] 值 [时间戳]`；
- 标签值：需转义`\`、`"`、`\n`；
- 值：支持普通浮点数、`+Inf`、`-Inf`、`NaN`。


### 2. 定义Prometheus的PEG语法
先拆解规则，从“根规则”到“最小单元”：
```cpp
peg::parser prom_parser(R"(
    # 根规则：文件由多个“注释行/指标行 + 换行”组成
    root        <- (commentline / vline) '\n'+
    # 注释行：区分HELP、TYPE、普通注释
    commentline <- ('# HELP ' name ' ' comment) / 
                   ('# TYPE ' name ' ' comment) / 
                   ('#' comment)
    # 指标行：带标签或不带标签
    vline       <- (name ' ' value (' ' timestamp)?) / 
                   (name labels ' ' value (' ' timestamp)?)
    # 标签集：{键=值, 键=值}
    labels      <- '{' nvpair (',' nvpair)* '}'
    # 标签键值对：name="label_value"
    nvpair      <- name '=' "\"" label_value "\""
    # 标签值：支持转义，直到遇到"
    label_value <- (!"\"" char)*
    # 字符：支持转义（\、"、\n）
    char        <- ('\\.' ) / (!'\\' .)
    # 辅助规则：注释内容（直到换行）、指标名（字母数字下划线）
    comment     <- (!'\n' .)*
    name        <- [a-zA-Z0-9_]+
    # 值：优先匹配+Inf/-Inf/NaN，再匹配普通浮点数
    value       <- '+Inf' / '-Inf' / 'NaN' / [0-9.+e-]+
    # 时间戳：整数（毫秒时间戳）
    timestamp   <- [+-]?[0-9]+
)");
```

这里有个关键设计：`value`规则把`+Inf`/`-Inf`/`NaN`放在前面。因为如果先匹配`[0-9.+e-]+`，`+Inf`会被拆成`+`和`Inf`，导致解析失败。PEG的“有序选择”正好解决了这个问题——先匹配特殊值，再匹配普通值。


### 3. 绑定语义动作：处理特殊值与转义
#### （1）处理指标值（含Inf/NaN）
Prometheus的`value`支持特殊浮点数，需要在语义动作里单独处理：
```cpp
#include <limits> // 用于获取Inf/NaN

prom_parser["value"] = [](const peg::SemanticValues& vs) {
    switch (vs.choice()) {
        case 0: return std::numeric_limits<double>::infinity(); // +Inf
        case 1: return -std::numeric_limits<double>::infinity(); // -Inf
        case 2: return std::numeric_limits<double>::quiet_NaN(); // NaN
        default: return vs.token_to_number<double>(); // 普通浮点数
    }
};
```

#### （2）处理标签值的转义
根据Prometheus规范，标签值只能转义`\`、`"`、`\n`，其他转义字符需报错：
```cpp
prom_parser["char"] = [](const peg::SemanticValues& vs) {
    std::string matched = vs.token_to_string();
    if (vs.choice() == 0) { // 转义分支
        char c = matched.at(1);
        // 检查是否为合规转义
        if (c != '\\' && c != '"' && c != 'n') {
            throw std::runtime_error(fmt::format("非法转义：\\{}", c));
        }
        // 把\n转成换行符
        return c == 'n' ? std::string(1, '\n') : std::string(1, c);
    }
    return matched; // 普通字符
};
```

#### （3）收集指标数据
最后定义一个`PromMetric`结构体，存储解析后的指标信息（名称、标签、值、时间戳、HELP/TYPE），并在语义动作里填充数据（完整代码可参考[网页示例](https://berthub.eu/articles/posts/practical-peg-parsing/)的`promparser.cc`）。


## 五、避坑指南：正则表达式不用“完美”
在PEG里用正则时，很多人会陷入“追求完美正则”的误区——比如为了匹配浮点数，写一个长达几十字符的正则（如`[-+]?[0-9]*\.?[0-9]+([eE][-+]?[0-9]+)?`）。但实际上，PEG的正则只需“匹配文本范围”，无需“完整验证格式”。

比如`Number <- [+-]?[0-9]*([.][0-9]*)?`会匹配`+`、`.`这类“非合法浮点数”，但我们可以用cpp-peglib的`predicate`钩子在解析后验证：
```cpp
// 给Number添加验证：确保是合法浮点数
parser["Number"].predicate = [](const peg::SemanticValues& vs) {
    std::string token = vs.token_to_string();
    // 尝试把token转成double，失败则返回false（解析报错）
    try {
        std::stod(token);
        return true;
    } catch (...) {
        return false;
    }
};
```

这样既简化了正则语法，又能通过“解析后验证”确保数据合法性，还能自定义错误信息（比如“无效数字：+”），比复杂正则更灵活。


## 六、cpp-peglib的进阶能力
除了基础解析，cpp-peglib还有很多实用特性，能应对更复杂的场景：
- **AST生成**：自动生成抽象语法树（AST），无需手动拼接数据，适合解析编程语言（如自定义脚本）；
- **增量解析**：支持流式数据解析（如从文件流、网络流中逐段解析），不用等完整数据；
- **工具支持**：提供在线语法调试工具（[cpp-peglib在线编辑器](https://yhirose.github.io/cpp-peglib/)），能实时查看匹配过程，快速定位语法错误；
- **跨语言参考**：如果不用C++，也可以用其他语言的PEG库——Go的`peggo`、Rust的`pest`、Python的`parsimonious`，核心逻辑与PEG一致。


## 总结：用PEG+cpp-peglib告别“糟糕的解析”
过去，写一个解析器需要权衡“复杂度”和“健壮性”——手动写的解析器易出错，Lex/Yacc又太复杂。而PEG+cpp-peglib打破了这个权衡：
- 简单场景（如向量、配置项）：几行代码就能实现，比正则更可靠；
- 复杂场景（如Prometheus、自定义协议）：200行代码就能覆盖转义、特殊值、错误处理，比手写代码少一半；
- 可扩展性：从基础解析到AST生成、流式解析，能随需求升级。

如果你下次再遇到“解析文本”的需求，不妨试试PEG+cpp-peglib——不用再靠正则“硬扛”，也不用为Lex/Yacc的配置头疼，用直观的规则就能写出健壮的解析器。