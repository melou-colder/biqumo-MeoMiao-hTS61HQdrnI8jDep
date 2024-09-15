
* wrktcp安装
码云地址：[https://gitee.com/icesky1stm/wrktcp](https://github.com)
直接下载，cd wrktcp\-master \&\& make,会生成wrktcp，就ok了，很简单
* wrktcp使用
压测首先需要一个服务，写了一个epoll\+边沿触发的服务，业务是判断ip是在国内还是国外，rq：00000015CHECKIP1\.0\.4\.0，rs：000000010，写的有些就简陋兑付看吧，主要为了压测和分析性能瓶颈。



```
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 


std::map<unsigned long, unsigned long> g_ip_list; // 存储 IP 范围

bool init_ip_list(const char* file_name, std::map<unsigned long, unsigned long> &ip_list)
{
    FILE *fp = nullptr;
    if ((fp = fopen(file_name, "r")) == nullptr)
    {
        return false;
    }

    int i = 0;
    int total_count = 0;
    char buf[64] = {0};

    while (fgets(buf, sizeof(buf), fp))
    {
        i++;
        if (buf[0] == '#')
            continue;

        char *pout = nullptr;
        char *pbuf = buf;
        char *pc[10];
        int j = 0;

        while ((pc[j] = strtok_r(pbuf, "|", &pout)) != nullptr)
        {
            j++;
            pbuf = nullptr;
            if (j > 7)
                break;
        }

        if (j != 7)
        {
            syslog(LOG_ERR, "%s:%d, unknown format the line is %d", __FILE__, __LINE__, i);
            continue;
        }

        if (strcmp(pc[2], "ipv4") == 0 && strcmp(pc[1], "CN") == 0)
        {
            unsigned long ip_begin = inet_addr(pc[3]);

            if (ip_begin == INADDR_NONE)
            {
                syslog(LOG_ERR, "%s:%d, ip is unknown, the line is %d, the ip is %s", __FILE__, __LINE__, i, pc[3]);
                continue;
            }
            int count = atoi(pc[4]);
            ip_begin = ntohl(ip_begin);
            unsigned long ip_end = ip_begin + count - 1;
            ip_list.insert(std::make_pair(ip_end, ip_begin));

            total_count++;
        }
    }

    syslog(LOG_INFO, "%s:%d, init_ip_list, total count is %d", __FILE__, __LINE__, total_count);

    fclose(fp);
    return true;
}

void extract_ip(char *buf, char *ip) {  
    // 假设协议字符串格式总是 "00000015CHECKIPx.x.x.x"  
    // 找到IP地址的起始位置  
    char *start = strstr(buf, "CHECKIP");  
    if (start == NULL) {  
        fprintf(stderr, "Invalid protocol string\n");  
        return;  
    }  
    // 跳过"CHECKIP"  
    start += 7;  
    // 复制IP地址到ip变量，注意检查边界  
    strncpy(ip, start, 15); // IP地址最多15个字符，包括'\0'  
    ip[15] = '\0'; // 确保字符串以'\0'结尾  
} 

// server
int main(int argc, const char* argv[])
{
	const char* file_name = "ip_list.txt";
    if (!init_ip_list(file_name, g_ip_list)) {
        std::cerr << "Failed to initialize IP list." << std::endl;
        return 1;
    }
	
    // 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(9999);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 本地多有的ＩＰ
    // 127.0.0.1
    // inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr.s_addr);
    
    // 设置端口复用
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    // 现在只有监听的文件描述符
    // 所有的文件描述符对应读写缓冲区状态都是委托内核进行检测的epoll
    // 创建一个epoll模型
    int epfd = epoll_create(100);
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(0);
    }

    // 往epoll实例中添加需要检测的节点, 现在只有监听的文件描述符
    struct epoll_event ev;
    ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
    ev.data.fd = lfd;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
    if(ret == -1)
    {
        perror("epoll_ctl");
        exit(0);
    }


    struct epoll_event evs[1024];
    int size = sizeof(evs) / sizeof(struct epoll_event);
    // 持续检测
    while(1)
    {
        // 调用一次, 检测一次
        int num = epoll_wait(epfd, evs, size, -1);
        printf("==== num: %d\n", num);

        for(int i=0; i// 取出当前的文件描述符
            int curfd = evs[i].data.fd;
            // 判断这个文件描述符是不是用于监听的
            if(curfd == lfd)
            {
                // 建立新的连接
                int cfd = accept(curfd, NULL, NULL);
                // 将文件描述符设置为非阻塞
                // 得到文件描述符的属性
                int flag = fcntl(cfd, F_GETFL);
                flag |= O_NONBLOCK;
                fcntl(cfd, F_SETFL, flag);
                // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
                // 通信的文件描述符检测读缓冲区数据的时候设置为边沿模式
                ev.events = EPOLLIN | EPOLLET;    // 读缓冲区是否有数据
                ev.data.fd = cfd;
                ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                if(ret == -1)
                {
                    perror("epoll_ctl-accept");
                    exit(0);
                }
            }
            else
            {
                // 处理通信的文件描述符
                // 接收数据
                char buf[128];
                memset(buf, 0, sizeof(buf));
                // 循环读数据
                while(1)
                {
                    int len = recv(curfd, buf, sizeof(buf)-1, 0);
                    if(len == 0)
                    {
                        // 非阻塞模式下和阻塞模式是一样的 => 判断对方是否断开连接
                        printf("客户端断开了连接...\n");
                        // 将这个文件描述符从epoll模型中删除
                        epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                        close(curfd);
                        break;
                    }
                    else if(len > 0)
                    {
                        // 通信
                        // 接收的数据打印到终端
                        write(STDOUT_FILENO, buf, len);
						char ip[16]; // 存储IP地址  
						extract_ip(buf, ip);  
						printf("Received IP: %s\n", ip);
						
                        // 发送数据
                        //send(curfd, buf, len, 0);
						// 验证 IP 地址
						struct in_addr address;
						int result = inet_pton(AF_INET, ip, &address); // 检查 IP 地址的有效性
						if (result < 0) {
							std::cout << "Invalid IP address: " << result << " " << ip << std::endl;
							send(curfd, "-Err\n", 5, 0);
							continue;
						}

						unsigned long ip_num = ntohl(address.s_addr);
						auto it = g_ip_list.lower_bound(ip_num);
						if (it != g_ip_list.end() && it->first >= ip_num && it->second <= ip_num) {
							send(curfd, "000000010", 9, 0); // 国内
						} else {
							send(curfd, "000000011", 9, 0); // 国外
						}
                    }
                    else
                    {
                        // len == -1
                        if(errno == EAGAIN)
                        {
                            printf("数据读完了...\n");
							close(curfd);
                            break;
                        }
                        else
                        {
                            perror("recv");
                            exit(0);
                        }
                    }
                }
            }
        }
    }

    return 0;
}

```

编译g\+\+ epoll\_test.cpp \-o epoll\_test,直接执行./epoll\_test,监听0的9999端口


* wrk配置文件sample\_tiny.ini



```
[common]
# ip & port
host = 127.0.0.1
port = 9999

[request]
req_body = CHECKIP1.0.4.0

[response]
rsp_code_location = head

```

说下其中的坑，req\_body就是要发的协议，但是wrktcp会在前面加长度固定8位：00000015；默认成功成功响应码是000000，设置rsp\_code\_location这个会让wrktcp从返回协议(000000010\)头开始找成功响应码
上面那些说明：wrktcp的README有一些说明，但解释的不太全，需要自己去试和看源码


* todo
固定协议前面加8位长度，不可能每个服务都是这样的协议，怎么去自定义的协议，希望大佬指教，好像wrk可以自定义协议。
* wrk压测命令
./wrktcp \-t15 \-c15 \-d100s \-\-latency sample\_tiny.ini



```
-t, --threads:     使用线程总数，一般推荐使用CPU核数的2倍-1
-c, --connections: 连接总数，与线程无关。每个线程的连接数为connections/threads
-d, --duration:    压力测试时间, 可以写 2s, 2m, 2h
--latency:     打印延迟分布情况
--timeout:     指定超时时间，默认是5000毫秒，越长占用统计内存越大。
--trace: 	   打印出分布图
--html: 	   将压测的结果数据，输出到html文件中。
--test:		   每个连接只会执行一次，一般用于测试配置是否正确。
-v  --version:     打印版本信息

```

测试了两遍，TPS能维持在1600左右



```
  Running 2m loadtest @ 127.0.0.1:9999 using sample_tiny.ini
  15 threads and 15 connections
  Time:100s TPS:1644.64/0.00 Latency:7.69ms BPS:14.45KB Error:0
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.66ms   14.17ms 318.09ms   98.89%
    Req/Sec   113.66    233.09     1.69k    94.95%
  Latency Distribution
     50%  823.00us
     75%    8.17ms
     90%    9.15ms
     99%   23.08ms
  164554 requests in 1.67m, 1.41MB read
Requests/sec:   1643.21    (Success:1643.21/Failure:0.00)
Transfer/sec:     14.44KB

```

* perf
压测监测服务:perf record \-p 10263 \-a \-g \-F 99 \-\- sleep 10
参数说明：
\-p : 进程
\-a : 记录所有事件
\-g : 启用基于 DWARF 调试信息的函数调用栈跟踪。这将记录函数调用栈信息，使得生成的报告更加详细，能够显示出函数调用的关系。
\-F : 采样频率
\-\-sleep：执行 sleep 命令，使系统休眠 10 秒钟。在这个期间，perf record 将记录指定进程的性能数据。


会在当前目录生成perf.data文件，执行perf report，会看到printf和write占用的CPU比较高，删除上面服务的printf和write函数，重新压测
![](https://img2024.cnblogs.com/blog/1249620/202409/1249620-20240914164622120-1495102010.png)
重新压测之后，TPS能维持在3W\+



```
Running 2m loadtest @ 127.0.0.1:9999 using sample_tiny.ini
  15 threads and 15 connections
  Time:100s TPS:32748.45/0.00 Latency:438.00us BPS:287.83KB Error:0
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   519.35us    1.24ms  63.18ms   97.47%
    Req/Sec     2.19k   536.83     4.83k    76.97%
  Latency Distribution
     50%  349.00us
     75%  426.00us
     90%  507.00us
     99%    5.12ms
  3275261 requests in 1.67m, 28.11MB read
Requests/sec:  32716.39    (Success:32716.39/Failure:0.00)
Transfer/sec:    287.55KB

```

 本博客参考[悠兔机场](https://xinnongbo.com)。转载请注明出处！
