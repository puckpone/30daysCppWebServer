# day18-HTTP有限状态转换机

为了实现`Http`服务器，首先需要对`Http`进行解析。对于`Http`请求报文。首先由四个部分组成，分别是请求行，请求头，空行和请求体组成。

其格式为
```
请求方法 URL HTTP/版本号
请求头字段
空行
body
```
例如
```
GET /HEELO HTTP/1.1\r\n
Host: 127.0.0.1:1234\r\n
Connection: Keep-alive\r\n
Content-Length: 12\r\n
\r\n
hello world;
```
可以看出，其格式是非常适合使用有限状态转换机。

为了实现这个功能，首先需要创建一个`HttpContext`解析器。这个解析器需要一个`HttpRequest`类来保存解析结果。
对于`HttpRequest`，他主要保存请求中的各种信息，如METHOD，URL等
```c++
class HttpRequest{
private:
    Method method_; // 请求方法
    Version version_; // 版本

    std::map<std::string, std::string> request_params_; // 请求参数

    std::string url_; // 请求路径

    std::string protocol_; // 协议

    std::map<std::string, std::string> headers_; // 请求头

    std::string body_; // 请求体
};
```

解析时，我们逐个字符遍历客户端发送的信息，首先设定`HttpContext`的初始状态为`START`。当我们遇到大写字母时，必然会是请求方法，因此此时，我们将此时的状态转成`METHOD`。当继续遇到大写字母时，说明`METHOD`的解析还在进行，而一旦遇到空格，说明我们的请求方法解析就结束了，我们使用`start`和`end`两个指针指向`METHOD`的起始位置和结束位置，获取相应的结果送入到`HttpRequest`中，之后更新`start`和`end`的位置，并更新当前的解析状态，进行下一个位置的解析。

```c++
class HttpContext
{
public:
    enum HttpRequestParaseState
    {
        kINVALID,         // 无效
        kINVALID_METHOD,  // 无效请求方法
        kINVALID_URL,     // 无效请求路径
        kINVALID_VERSION, // 无效的协议版本号
        kINVALID_HEADER,  // 无效请求头

        START,  // 解析开始
        METHOD, // 请求方法

        BEFORE_URL, // 请求连接前的状态，需要'/'开头
        IN_URL,     // url处理
        ...
        COMPLETE, // 完成
    };
    // 状态转换机，保存解析的状态
    bool ParaseRequest(const char *begin, int size);

private:
    std::unique_ptr<HttpRequest> request_;
    HttpRequestParaseState state_;
};

// HttpContext.cpp
bool HttpContext::ParamRequest(const char *begin, int size){
    //
    char *start = const_cast<char *>(begin);
    char *end = start;
    while(state_ != HttpRequestParaseState::kINVALID 
        && state_ != HttpRequestParaseState::COMPLETE
        && end - begin <= size){
        
        char ch = *end; // 当前字符
        switch(state_){
            case HttpRequestParaseState::START:{
                if(isupper(ch)){
                    state_ = HttpRequestParaseState::METHOD;
                }
                break;
            }
            case HttpRequestParaseState::METHOD:{
                if(isblank(ch)){
                    // 遇到空格表明，METHOD方法解析结束，当前处于即将解析URL，start进入下一个位置
                    request_->SetMethod(std::string(start, end));
                    state_ = HttpRequestParaseState::BEFORE_URL;
                    start = end + 1; // 更新下一个指标的位置
                    }
                break;
            }
            case HttpRequestParaseState::BEFORE_URL:{
                // 对请求连接前的处理，请求连接以'/'开头
                if(ch == '/'){
                    // 遇到/ 说明遇到了URL，开始解析
                    state_ = HttpRequestParaseState::IN_URL;
                }
                break;
            }
            ...
        }
        end ++;
    }
}
```

举例说明工作流程：
假设客户端发送的请求内容如下（用 \r\n 表示换行）：

  "GET /index.html?user=alice&age=30 HTTP/1.1\r\nHost: example.com\r\n\r\n"

解析过程大致如下：

解析 METHOD
 - 初始状态为 START，当解析到第一个非空字符 'G'（大写字母）时，状态转为 METHOD。
 - 当遇到空格时（此时 start 指向 'G'，end 指向空格），执行
  std::string(start, end) 得到 "GET"。
 - 调用 request_->SetMethod("GET") 存入 HttpRequest 对象中，并将 state 切换到解析 URL 的状态。

解析 URL
 - 状态变为 BEFORE_URL 后，当遇到 '/' 字符时，状态切换到 IN_URL。
 - 在 IN_URL 状态中，指针从 '/' 开始向后扫描。当遇到 '?' 时，说明 URL 的路径结束且后面有参数。此时 std::string(start, end) 得到 "/index.html" 被存入 HttpRequest 对象中（通过 SetUrl 函数）。
 - 然后状态切换到处理 URL_QUERY 参数，start 指针移动到 '?' 后的下一个字符。

解析 URL 请求参数
 - 在 URL_PARAM_KEY 和 URL_PARAM_VALUE 状态中，系统遇到 '='、'&' 或空格时，就会使用 std::string(start, colon) 得到参数 key，再使用 std::string(colon + 1, end) 得到参数 value。
 - 对于本例，首先会解析出参数 "user" 对应 "alice"，当遇到 '&' 时状态切换并存入 map 中。随后继续解析，得到 "age" 对应 "30"。

解析 HTTP 协议与版本
 - 当解析到空格后，进入 BEFORE_PROTOCOL 状态，然后在 PROTOCOL 状态中，当遇到 '/' 时通过 std::string(start, end) 得到 "HTTP"（或“HTTP1.1”的一部分），接着在 VERSION 状态中继续解析版本号，最终调用 SetVersion 存入 HttpRequest 对象。

解析 Header
 - 紧接着遇到 \r\n，状态进入 HEADER_KEY/HEADER_VALUE 状态。例如，当读取 "Host: example.com" 时，std::string(start, colon) 得到 "Host"，std::string(colon + 2, end) 得到 "example.com"。存入 HttpRequest 的 headers_ map 中。

解析结束与请求体
 - 当连续遇到空行（\r\n\r\n），状态转为 COMPLETE，解析结束（本例没有请求体）。

整个解析过程中，std::string(start, end) 的作用就是提取从当前指针 start 开始，到当前指针 end（不包括 end 指向字符）之间的一段字符，转换成 std::string。

这样最终构建的 HttpRequest 对象里面存放了：
 - method: "GET"
 - url: "/index.html"
 - 请求参数：{ "user" : "alice", "age" : "30" }
 - 协议版本等信息
 - headers：{ "Host" : "example.com" }


实际上这个版本并不鲁棒，而且性能并不是十分优异，但是作为一个玩具而言，并且去熟悉HTTP协议而言，还是有些价值的。

在本日额外实现了一个`test_httpcontext.cpp`文件用于测试`HttpContext`的效果。在`CMakeLists.txt`加入`http`的路径，并创建`build`文件，在`build`中运行`cmake ..`，之后`make test_context`并运行`./test/test_contxt`即可。

特别的，感谢[若_思CSDN C++使用有限状态自动机变成解析HTTP协议](https://blog.csdn.net/qq_39519014/article/details/112317112)本文主要参考了该博客进行了修改和实现

