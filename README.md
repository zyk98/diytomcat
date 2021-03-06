# Mini-Tomcat
## 简介
本项目内容是一个简化版的Tomcat服务器。

此服务器主要是仿照Tomcat的源码,利用**Java基础、反射、log4j、junit、jsoup、hutool**等知识进行构建。

涉及到的主要功能包括:
    **构建Tomcat内置对象，处理多种文件格式(包括二进制文件)，处理Servlet请求，处理Jsp请求，客户端跳转，过滤器，文件部署以及监听器**的功能。

![10584](https://gitee.com/zyk98/images/raw/master/img/10584.png)

## Tomcat内置对象
   - ### Request
      Request对象主要用于获取请求的相关信息uri、requestString、头信息、请求参数   
   - ### Response
        Response用于封装返回给浏览器的数据
   - ### Context
     
      每个Context实例代表了一个应用,用于多应用的加载
        - 属性：path(访问路径),docBase(文件夹所在的绝对路径)
      
        - 加载时机：在服务器启动的时候把webapps目录下的文件夹加载成Context对象了
      
        - 解析时机：在构造Request的时候就把Context解析出来
      
        ```
        String fileName = StrUtil.removePrefix(uri, "/");
        File file = FileUtil.file(context.getDocBase(), fileName);
      ````
      
        优势：将uri中访问的资源与实际服务器资源对应起来，代码如上
        1. 通过配置的方式来访问Context
            主要是使用XML文件来进行配置，使用Jsoup进行解析，遍历所有的Context标签的将里面的内容封装到Context
        2. 扫描webapp目录下面的文件夹然后进行context装载

   - ### Host    
        代表虚拟主机，每个Host下面有多个Context因为Host的下一层是Context所以将
        
        scanContextsOnWebAppsFolder();：用于解析Webapp目录下下面的文件夹导入ContextMap
        
        scanContextsInServerXML();：解析配置文件中的内容并装入Context中
        
        放在Host的构造方法中。
        ```
        public Host(String name, Engine engine){
            this.contextMap = new HashMap<>();
            this.name =  name;
            this.engine = engine;
            scanContextsOnWebAppsFolder();
            scanContextsInServerXML();
        }
        ```
   - ### Engine
        代表Servlet引擎，每个Engine下面有多个Host
        
        通过解析XML文件获得Engine中默认虚拟主机和所有主机
        
        ```
        public Engine(Service service) {
            this.service = service;
            this.defaultHost = ServerXMLUtil.getEngineDefaultHost();
            this.hosts = ServerXMLUtil.getHosts(this);
            checkDefault();
        }
        ```
   - ### Connector
        代表连接的不同连接点，一个Service一般对应多个Connector端口
        
        构造方法中要加入Service，启动服务器套接字，之后使用多线程来处理客户端的连接请求
        ```
        ServerSocket s = new ServerSocket(port);
        while(true){
            //服务器随时接受请求并分配一个线程工作
            Socket s = ss.accept();
            Runnable r = new Runnable(){
                @Override
                public void run(){
                    //操作
                }
            };
            ThreadPoolUtil.run(r);
        }
        ```
   - ### Service
        代表Tomcat提供的服务，一个Service对应一个Engine，一个Service对应多个连接点
        ```
        public Service(Server server){
            this.server = server;
            this.name = ServerXMLUtil.getServiceName();
            this.engine = new Engine(this);
            this.connectors = ServerXMLUtil.getConnectors(this);
        }
        ```
   - ### Server
        打印java虚拟机相关的信息，然后初始化调用Service中的start方法
        ```
        public void start(){
            TimeInterval timeInterval = DateUtil.timer();
            logJVM();
            init();
            LogFactory.get().info("Server startup in {} ms",timeInterval.intervalMs());
        }
        ```
### 多种文件格式
   - 非二进制文件
      使用WebXMLUtil类进行解析，使用Map结构将所有的类型进行保存，之后根据类型名获取相应的类型。为了防止多次初始化，使用了Synchronized进行同步
   - 二进制文件
      Response使用字节数组存放二进制文件，增加set与get方法。在Server中修改读取文件的方式，直接读取定位到的文件字节数组，之后封装到Response中

### 处理Servlet

   - #### 配置Servelt功能
       - 在Context类中存储访问路径与Servlet的对应关系
       
        ```
        //url对应的Servlet类名
        private Map<String, String> url_servletClassName;
        //url对应的Servlet名称
        private Map<String, String> url_servletName;
        //Servlet名称对应的类名
        private Map<String, String> servletName_className;
        //Servlet类名对应的名称
        private Map<String, String> className_servletName;
        ```
        - 在InvokerSetvlet中通过反射来调用Service方法

   - #### 类加载器
        - Tomcat类加载体系
            顶层是公共类加载器CommonClassLoader负责的是%tomcat_home%/lib目录的类和jar
            下面是WebappClassLoader用于加载某个web应用
            之后是JspClassLoader用于加载jsp转化为.java被编译的类
        - 公共类加载器
            继承了URLClassLoader扫描lib包下面的类和jar，将路径转化为file:xxx
        - web应用类加载器
            每个web应用都包含私有的WebClassLoader。
            主要是扫描Context对应的docBase下面的classes和lib。其中classes目录作为URL加入，结尾要加"/"。

            ```
            ClassLoader commonClassLoader = Thread.currentThread().getContextClassLoader();
            this.webappClassLoader = new WebappClassLoader(docBase, commonClassLoader);
            ```
            上面代码在构造方法中进行初始化，先获得Bootstrap里的commonClassLoader之后再初始化webappClassLoader 
        - #### InvokerServlet处理Servlet
        - 设计为单例设计模式
        - 处理Servlet的流程
        ```
        //得到uri
        String uri = request.getUri();
        //得到请求中的context
        Context context = request.getContext();
        //得到Servlet对应的ServletClass
        String servletClassName = context.getServletClassName(uri);
        Class servletClass = context.getWebappClassLoader().loadClass(servletClassName);
        Object servletObject = context.getServlet(servletClass);
        ReflectUtil.invoke(servletObject, "service", request, response);
        
        ```
   - #### DefaultServlet处理静态资源
        - 采用单例设计模式
   - #### 热加载
        - 概念
        当web项目下面的Classes目录下面的资源发生变化，或者是lib里面的jar发生变化，就会重新加载当前的Context
        - 流程
            - 先创建Context
            - 创建专属监听器，用于监听docBase下的文件变化
            - 持续监听
            - 判断发生变化的文件的后缀名           
                 1. 不是class，jar或者是xml继续监听
                 2. 否则，关闭监听，重载Context，然后刷新Host里面的contextMap
            - 重载意为新建一个Context    
   - #### Servlet对象
        - ServletContext
        创建了一个attributesMap用于存放属性，其中内置一个context。ApplicationContext 的很多方法，其实就是调用的是context。
        - ServletConfig
        是在Servlet初始化的时候，传递进去的参数对象
        - Servlet单例
        实现单例的方法：在Context中初始化一个servletPool，每次访问servlet都会根据servletClass对象都去池子中取。其中InvokerServlet
        获取servletObject对象都是从池子中获取
        - Servlet生命周期
        实例化、初始化、提供服务、销毁、被回收
        其中销毁的时候先关闭类加载器，再关闭监听器，最后销毁Servlet
        - Servlet自启动
        在Context创建需要自启动的servlet类名，之后再初始化的时候进行自启动
        - Cookie
        Cookie是在服务端创建的，保存在浏览器端，用于数据交互。
        服务器端如何接受Cookie：获取headerMap中的cookie，解析Cookie存入List中。
        - Session
        检测session是否有效，使用一个线程默认30s检测一次，如果失效就从sessionMap中去除
        
        SessionManager获取Session的逻辑
        
        1. 先判断jsessionid是否存在，不存在新建一个session
        2. 如果jsessionid无效，那么新建一个sessionid
        3. 否则使用现有的session并修改lastAccessedTime，创建响应的cookie
         - #### JSP对象
        - jsp转译
        先将jsp转译成.java文件，再将.java文件编译成.class文件
        转译的jsp文件继承了HttpJspBase又继承了JspPage类最后继承了HttpServlet类，本质上还是一个Servlet
        - 编译时机
        先判断jsp是否存在 否 返回404
        是 是否有对应的class文件 否 生成class
        是 jsp文件是否晚于class文件 否 直接使用
        是 重新生成class
        - JspClassLoader
        1. 一个jsp文件对应一个JspClassLoader
        2. 如果jsp文件修改需要换一个新的JspClassLoader
        3. JspClassLoader基于jsp文件转译并编译出来的class文件，进行类加载
        - #### 跳转功能
        - 客户端跳转

    302临时跳转
    - 服务端跳转
    请求转发功能即服务器内部修了request的uri，然后再执行一次HttpProcessor中的execute方法。

### 过滤器
 - 责任链模式
    使多个对象都有机会处理请求，从而避免请求的发送者和处理者之间的耦合关系，将这和对象连成一条链并沿着
    这条链传递请求，直到有一个对象来处理它
 - 在Context中新建一系列跟过滤器加载和初始化的参数
    ```
    private Map<String, List<String>> url_filterClassName;
    private Map<String, List<String>> url_FilterNames;
    private Map<String, String> filterName_className;
    private Map<String, String> className_filterName;
    private Map<String, Map<String, String>> filter_className_init_params;
    ```
### war部署
 - 静态war部署
 将war文件解压为目录并把文件夹加载为Context
 - 动态war部署
 用监听器监听webapps目录，当新创建war文件就调用静态部署中的方法
### 监听器模式
 - Context变动进行监听

