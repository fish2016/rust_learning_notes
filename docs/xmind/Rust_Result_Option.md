unwrap和expect的区别是什么？  它们都会在否定条件下触发panic

expext的语义是，期望有值（就是期望非空），否则返回panic携带错误信息
Result中，使用unwrap触发panic会有自定义错误信息；使用expect触发panic显示的是expect的参数错误信息，而不是Err()的
Option中，使用unwrap触发panic无自定义错误信息；使用expect会显示自定义错误信息


unwrap    语义是解包(打开包装),解析出Result，如果能解析出OK，则返回，解析不出则panic； 所以unwrap的语义可以理解为解包出某个值
unwrap_or 解析出Result，如果是OK，则返回，如果不是OK，则返回or后面的值


unwrap\expect\ match \if let
- 不建议直接使用 unwrap ，因为它可能导致程序崩溃
- 建议使用更安全的替代方法：
- unwrap_or(默认值)  //否的情况下，返回默认值，不触发panic
- unwrap_or_else(|| 计算默认闭包) //否的情况下，执行默认闭包（闭包是有返回值的），不触发panic
- unwrap_or_default() //返回值类型的零值作为默认值，比如int的零值是0， string的零值是空串""
- expect("错误信息")
- 使用 match 或 if let 模式匹配
- match if 模式匹配
match和if let的区别
match关注多个返回值模式
if let只关注一个返回值模式

match + if 混合判断（也称 "match guard"）

unwrap 在 Rust 中是一个用于从 Result 或 Option 类型中提取值的方法。它的语义是：

1.  对于 Result<T, E> ：
   
   - 如果是 Ok(value) ，返回 value
   - 如果是 Err(e) ，程序会 panic
2. 对于 Option<T> ：
   
   - 如果是 Some(value) ，返回 value
   - 如果是 None ，程序会 panic（崩溃）
   