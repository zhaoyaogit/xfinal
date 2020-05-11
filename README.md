# xfinal
一个用于快速使用c++ 开发web服务器的框架

# xfinal简介
高性能易用的http框架,基于c++11标准开发

1. 统一而简单的接口
2. header-only
3. 跨平台
4. 支持拦截器功能
5. 支持session/cookie （持久化,默认保存到磁盘。用户可以自定义持久化介质）
6. 支持websocket 
7. 易用的客户端(http client/ https client 可以通过CMakeList.txt中的ENABLE_CLIENT_SSL ON开启)

# 目录
* [如何使用](#如何使用)
* [快速示例](#快速示例)
* [xfinal配置](#功能配置)
* [RoadMap](#RoadMap)
* [联系方式](#联系方式)


# 如何使用

## 编译依赖
使用支持c++11标准的c++ 编译器即可编译运行,开箱即用,无需其他依赖(linux 下只需要uuid库，大部分linux系统默认都有)
### Linux uuid安装
##### centos
sudo yum install libuuid-devel.x86_64  
##### ubuntu
sudo apt-get install uuid  
sudo apt-get install uuid-dev

## 生成编译
mkdir build  
cd build  
cmake ..  
即可生成相应平台下的编译项目  


# 快速示例

## GET请求 服务器返回Hello,world
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/hello",[](request& req,response& res){
      res.write_string("hello,world");
   });
   serve.run();
}
````
## GET请求 url参数获取
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/param",[](request& req,response& res){
      auto id = req.param("id");
      res.write_string(view2str(id));
   });
   serve.run();
}
````

## GBK的支持
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET,POST>("/url",[](request& req,response& res){
      auto id = req.param<GBK>("id"); /*提供GBK转码的支持*/
      auto name = req.query<GBK>("name"); /*提供GBK转码的支持*/
      res.write_string("GBK support");
   });
   serve.run();
}
````
## URL 重定向
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/redirect",[](request& req,response& res){
       res.redirect("/abc",false); /*接口的第二个参数用来告诉框架是临时重定向 还是永久重定向 */
   });
   serve.run();
}
````

## url form 表单请求
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/form",[](request& req,response& res){
        auto name = req.query("name");  /*query接口用于查询 name="xxx" 的值*/ 
        res.write_string(view2str(name));
   });
   serve.run();
}
````
## multipart form 表单请求
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/form",[](request& req,response& res){
        auto name = req.query("name");  /*query接口用于查询 name="xxx" 的值 */ 
        auto& file = req.file("img"); /*file 接口用于查询 name="xxx" 的文件 */ 
        res.write_string(std::to_string(file.size()));
   });
   serve.run();
}
````
## octet-stream 请求处理

````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/form",[](request& req,response& res){
        auto& file = req.file();  /*octet-stream 只能提交单个二进制流数据 所以这里的file没有参数 */
        res.write_string(std::to_string(file.size()));
   });
   serve.run();
}
````
## 支持泛化url重载 
#### 自动路由到最佳匹配的url路径 { /* , /abc/* , /abc/ccc/* , /abc   }
>请求示例  
>> 1. url为 **http://127.0.0.1:8080/abc/ccc/ddd**,则最佳匹配的是**/abc/ccc/* **  
>> 2. url为 **http://127.0.0.1:8080/abc/cccc/ddd**, 则匹配** /abc/* **,  
>> 3. url为 **http://127.0.0.1:8080/ddd/kkk,则只匹配 **/* **
>> 4. url为 **http://127.0.0.1:8080/abc**,则最佳匹配的就是**/abc **
````cpp
#include <xfinal.hpp>
using namespace xfinal;

int main()
{
	auto const thread_number = std::thread::hardware_concurrency();
	http_server server(thread_number);
	bool r = server.listen("0.0.0.0", "8080");
	if (!r) {
		return 0;
	}
	server.set_url_redirect(false);
	
	server.router<GET>("/*", [](request& req, response& res) {
		auto url = req.raw_url();
		json root;
		root["router"] = "/*";
		root["url"] = view2str(url);
		auto params = req.url_params();
		for (auto& iter : params) {
			root["params"].push_back(view2str(iter));
		}
		root["param_size"] = req.url_params().size();
		res.write_json(root);
	});


	server.router<GET>("/abc", [](request& req, response& res) {
		res.write_string("abc");
	});

	server.router<GET>("/abc/*", [](request& req, response& res) {
		auto url = req.raw_url();
		json root;
		root["router"] = "/abc/*";
		root["url"] = view2str(url);
		auto params = req.url_params();
		for (auto& iter : params) {
			root["params"].push_back(view2str(iter));
		}
		root["param_size"] = req.url_params().size();
		res.write_json(root);
	});


	server.router<GET>("/abc/ccc/*", [](request& req, response& res) {
		auto url = req.raw_url();
		json root;
		root["router"] = "/abc/ccc/*";
		root["url"] = view2str(url);
		auto params = req.url_params();
		for (auto& iter : params) {
			root["params"].push_back(view2str(iter));
		}
		root["param_size"] = req.url_params().size();
		res.write_json(root);
	});
	
	server.run();
	return 0;
}
````
## 返回json
#### 使用了第三方json库 更多用法参考 [nlohmann/json](https://github.com/nlohmann/json)
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/json",[](request& req,response& res){
      json data;
      data["name"] = "xfinal";
      data["language"] = "morden c++";
      res.write_json(data); /*第一个参数也可以直接传入json字符串,第二个接口用来控制返回方式*/
   });
   serve.run();
}
````

## 视图层 用于前端模板渲染
#### 视图层更多使用方法 参考 [xmh0511/inja](https://github.com/xmh0511/inja) （在inja库的基础上修改并增添了功能）
````cpp
///test.html
<!DOCTYPE html>

<html lang="zh" xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta charset="utf-8" />
    <title></title>
</head>
<body>
    <div style="color:aqua">
        <p><span>name:</span>@{name}</p>
        <p><span>language:</span>@{language}</p>
        <p>#{ @{language} }#</p>  <!--原样输出-->
	<a href="https://www.baidu.com">@{ str2int("30")}</a>
        <p>@{ str2int(str)}</p>
    </div>
</body>
</html>
///test.html

#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   server.add_view_method("str2int", 1, [](inja::Arguments const& args) {
	auto i = std::atoi(args[0]->get<std::string>().data());
	return std::string("transform:") + std::to_string(i);
   });
   serve.router<GET>("/view",[](request& req,response& res){
       res.set_attr("name", "xfinal");
       res.set_attr("language", "c++");
       res.write_view("./static/test.html");
   });
   serve.run();
}
````

## 文件请求
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   server.add_view_method("str2int", 1, [](inja::Arguments const& args) {
	auto i = std::atoi(args[0]->get<std::string>().data());
	return std::string("transform:") + std::to_string(i);
   });
   
   serve.router<GET>("/file",[](request& req,response& res){
      res.write_file("./static/need.bmp",/*这里有个bool类型参数 可以用来告诉接口 是直接返回文件内容(false) 还是以chunked的方式返回文件  
      (true)*/);
      /*chunked 方式支持断点续传 */
      /*框架内置的静态文件处理提供了通用的处理能力，这里提供了让用户处理更多资源文件请求的可能性，比如通过参数进行图片视频等多媒体文件的裁剪*/
   });
   serve.run();
}
````
## xfinal 框架内置了静态文件处理
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   /*可以通过serve.static_path("./static") 改变目录 同时会自动改变静态资源的url的标识*/
   /*内置处理方式默认设置了static  用户可以通过localhost:8080/static/abc.png 来访问静态资源文件*/
   /*对于多媒体等文件默认支持断点续传功能*/
   serve.run();
}
````
## xfinal session/cookie 支持
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<GET>("/login",[](request& req,response& res){
       req.create_session().set_data("islogin",true);  /*可以通过设置参数指定session名称 req.create_session("myname")*/
       res.write_string("OK");
   });
   serve.router<GET>("/checklogin",[](request& req,response& res){
        auto& session = req.session();  /* req.session("myname") */
	if(session.get_data<bool>("islogin")){
	    res.write_string("yes");
	}else{
	  res.write_string("no");
	}
   });
   serve.run();
}
````
### 用户可以自定义持久化介质
````cpp
#include <xfinal.hpp>
using namespace xfinal;
class sql_storage:public session_storage{
   public:
       bool init(){
       /*初始化操作，从介质读取数据*/
      }
      void save(session& session){
        /*用来保存session*/
      }
      void remove(session& session){
         /*用来移除session*/
      }
      void remove(std::string const& uuid){
        /*通过uuid 删除session*/
      }
};
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.set_session_storager<sql_storage>(); /*设置session存储方式*/
   serve.run();
}
````
## xfinal 为你的接口提供拦截器
````cpp
#include <xfinal.hpp>
using namespace xfinal;
struct http_interceptor{
   bool before(request& req,response& res){  //这里的返回值用来告诉框架是否还继续执行用户接下来的拦截器以及注册的路由逻辑
       auto id = req.query("id");
       if(id=="0"){
          return true;
       }else{
         return false;
       }
   }
   
   bool after(request& req,response& res){
      res.add_header("Access-Control-Allow-Origin","*");
      return true;
   }
};
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<POST>("/interceptor",[](request& req,response& res){
       res.write_string("OK");
   },http_interceptor{}); //可以注册多个拦截器 定义如示例中的http_interceptor
   serve.run();
}
````
## xfinal 请求拦截处理
````cpp
#include <xfinal.hpp>
using namespace xfinal;
struct pre_interceptor{
   bool prehandle(request& req,response& res){  //这里的返回值用来告诉框架是否继续处理当前http请求，如果返回false则返回用户通过write_xxx的信息，并断开连接
       if(req.content_type() == content_type::multipart_form){  //如果是上传文件 则断开连接
          res.write_string("invalid request",http_status::bad_request );
          return false;
       }
       return true;
   }
   
   bool posthandle(request& req,response& res){
      return true;
   }
};
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   serve.router<POST>("/interceptor",[](request& req,response& res){
       res.write_string("OK");
   },pre_interceptor{}); //可以注册多个拦截器 定义如示例中的http_interceptor
   serve.run();
}
````
## xfinal 异常错误信息获取
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main(){
  http_server serve(4); //线程数
  serve.listen("0.0.0.0","8080");
  serve.on_error([](std::exception const& ec) {  //提供用户记录错误日志
       std::cout << ec.what() << std::endl;
  });
  serve.run();
}
````
## xfinal 支持项目结构分层
### 自定义class 
````cpp
////Test.hpp
#pragma once
#include <xfinal.hpp>
class Test{
  public:
    void shop(request& req,response& res){
        res.write_string("shop !");
    }
} ;
////Test.hpp
#include "http_server.hpp"
#include "Test.hpp"
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   /*不保存接口状态信息*/
   serve.router<GET,POST>("/shop",&Test::shop,nullptr);
   
   /*这里可以记录下每次请求的一些信息，用于下次请求使用*/
   Test t;
   serve.router<GET,POST>("/shop",&Test::shop,t);
   serve.run();
}
````
### 继承xfinal 的Controller
````cpp
///shop.hpp
#pragma once
#include <xfinal.hpp>
class Shop:public Controller{
  public:
    void go(){
        this->get_response().write_string("go shopping!");
    }
};
///shop.hpp

#include "http_server.hpp"
#include "shop.hpp"
using namespace xfinal;
int main()
{
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");
   /*不保存接口状态信息*/
   serve.router<GET,POST>("/shop",&Shop::go,nullptr);
   
    /*这里可以记录下每次请求的一些信息，用于下次请求使用*/
   Shop t;
   serve.router<GET,POST>("/shop",&Shop::go,t);
   serve.run();
}
````
### 以上所有使用lambda注册的路由 都可以替换成成员函数或controller

# 好用的webscoket
````cpp
#include <xfinal.hpp>
int main()
{
    http_server serve(4); //线程数
    serve.listen("0.0.0.0","8080");
    
    	websocket_event event;  //定义websocket 事件
	event.on("message", [](websocket& ws) {
		std::cout << ws.messages() << std::endl;  //消息内容
		std::cout << (int)ws.message_code() << std::endl; //消息类型
		std::string message;
		for (auto i = 0; i <= 400; ++i) {
			message.append(std::to_string(i));
		}
		ws.write_string(message);  //发送消息 ws.write_binary("xxx")用来写二进制类型的数据 ||  ws.write("xxx",opcode);
	}).on("open", [](websocket& ws) {
		std::cout << "open" << std::endl;
	}).on("close",[](websocket& ws){
	   //balabala
	});
	//可能做聊天工具等 通过ws.shared_from_this() 保存当前会话 自行发挥 ^_^
	server.router("/ws", event);  //定义webscoket 路由
}
````
# 功能配置
>如果没有特殊说明,默认在run方法调用前进行设置
### 设置是否重定向url
> 在url请求中,请求的地址是 http://1270.0.1:8080/abc/ 如果`server.set_url_redirect(true);`将会重新跳转，将地址设置为`http://127.0.0.1:8080/abc`
````cpp
server.set_url_redirect(false);
````
### 设置静态文件读取路径（会自动设置upload路径到当前静态路径的upload文件夹中）
> 调用此接口后，请求的地址 http://127.0.0.1:8080/x/y/somefile.abc 就会自动到该目录下读取对应的文件
````cpp
server.set_static_path("./x/y");
````
### 单独设置上传文件保存的路径
> 在set_static_path后进行设置
````cpp
server.set_upload_path("./myupload");
````
### 设置socket读写超时时间
>单位: 秒 超时关闭当前socket连接
````cpp
server.set_wait_read_time(10);
````
### keep-alivek空闲时长
>单位:秒  超时自动断开
````cpp
server.set_keep_alive_wait_time(5);
````
### 设置session有效速率（每个x秒检测session是否有效）
>单位:秒
````cpp
server.set_check_session_rate(10);
````
### 设置websocket返回数据每帧大小
>单位：字节
````cpp
server.set_websocket_frame_size(1024 * 1024);
````
### 设置websocket空闲时长
>单位：秒 超时自动断开
````cpp
set_websocket_check_alive_time(3600)
````
#### 以上方法去除set_ 就能获取对应的配置数据

### 设置服务器错误信息的处理
>需要在listen方法调用前设置  
````cpp
server.on_error([](std::string const& message) {  //提供用户记录错误日志
	std::cout << message << std::endl;
});
````
### 设置全局404处理方法
>可以提供用户自己处理404需要返回的内容，如404个性化页面等
````cpp
server.set_not_found_callback([](request& req,response& res) {
	res.write_string("custom not found", true, http_status::bad_request);
});
````

### 为模板引擎增加方法
````cpp
server.add_view_method("str2int", 1, [](inja::Arguments const& args) {
	auto i = std::atoi(args[0]->get<std::string>().data());
	return std::string("transform:") + std::to_string(i);
});
// 比如在html 中可以使用 @{str2int("1024")} 
````

# xfinal client 使用
## GET请求
### client 同步
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main(){
   http_server serve(4); //线程数
   serve.listen("0.0.0.0","8080");

   serve.router<GET>("/client",[](request& req,response& res){
        http_client client("www.baidu.com");  //或者IP:端口号
        auto con = client.request<GET>("/");  //请求的url
		  auto header = con.get_header("Content-Length");  //获取响应头信息
		  auto state = con.get_status_code();  //获取响应状态码
		  res.write_string(con.get_content());  //获取响应的body
   });
   serve.run();
}
````
### client 异步
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main(){
      http_server serve(4); //线程数
      serve.listen("0.0.0.0","8080");
      http_client client("www.baidu.com");

      server.router<GET>("/asyclient", [](request& req, response& res) {
		http_client client("www.baidu.com");
         client.request<GET>("/", [](http_client::client_response const& res,std::error_code const& ec) {
            if (ec) {
               return;
            }
            std::cout << res.get_content() << std::endl;
         });
         client.run();
         res.write_string("OK");
	   });

     serve.run();
}
````
## POST请求
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main(){
      http_server serve(4); //线程数
      serve.listen("0.0.0.0","8080");
      server.router<GET>("/client_post", [](request& req, response& res) {
         try {
            http_client client("127.0.0.1:8080");
            client.add_header("name", "hello");
            client.add_header("Content-Type", "application/x-www-form-urlencoded");
            std::string form = "id=1234";
            auto con = client.request<POST>("/test", form);
            res.write_string(con.get_content());
         }
         catch (...) {
            res.write_string("error");
         }
	   });

      serve.run();
}
````
## POST multipart form
````cpp
#include <xfinal.hpp>
using namespace xfinal;
int main(){
      http_server serve(4); //线程数
      serve.listen("0.0.0.0","8080");
	   server.router<GET>("/client_multipart", [](request& req, response& res) {
         try {
            http_client client("127.0.0.1:8020");
            multipart_form form;
            form.append("img", multipart_file{ "./static/upload/v2.jpg" });
            form.append("text", "hello,world");
            auto con = client.request<POST>("/upload", form);
            res.write_string(con.get_content());
         }
         catch (...) {
            res.write_string("error");
         }
	   });
      serve.run();
}
````
## POST client 异步
````cpp
同上 request 传入回调函数
````
# xfinal client 同时支持https

# RoadMap
1.提升xfinal的稳定性  
2.优化xfinal性能  

# 联系方式
QQ&ensp;:&ensp;970252187  
Mail&ensp;:&ensp;970252187@qq.com  

# Donation  
如果项目对你有所帮助，请给作者一点鼓励和支持  
### alipay:  
![image](https://github.com/xmh0511/donation/blob/master/alipay.png)  
### wechat:  
![image](https://github.com/xmh0511/donation/blob/master/wechat.png)  
