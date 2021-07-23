## Linux高性能服务器编程

### 1、Linux网络编程API

1、主机字节序和网络字节序

小端序：高位字节存放高位地址

一般的地址的顺序是按照从小到大的顺序，字节的顺序在大多数PC中都是小端序

大端序：高位字节存放到底位地址

例如：0x12345678，在大端序中表示就是0x12345678，小端序中表示就是0X78563412

网络中一般采用大端序，因此需要进行转换

```c
#include<netinet/in.h>
unsigned long int htonl(unsigned long int hostlong)  //将主机字节序转换为网络字节序
unsigned short int htons(undigned short int hostshort)  //将主机的字节序转换为网络字节序
unsigned long int ntohl(unsigned long int netlong)  //将网络字节序转换为主机字节序
unsigned short int ntohs(unsigned short int netshort)  //将网络字节序转换为主机字节序
```

一般长整型用来对IP地址进行转换，短整型用来对端口号进行转换

2、通用socket地址

```c
//IPV4
struct sockadrr_in
{
	sa_family_t sin_family;  //地址族：AF_INET
	u_int16_t sin port;  //端口号，用网络字节序表示
	struct in_addr sin_addr;  //IPV4地址结构
};
struct in_addr
{
	u_int32_t s_addr;  //ipv4地址，用网络字节序表示
}
```

3、ip地址转换函数

```c
#include<arpa/inet.h>
in_addr_t inet_addr(const char * strptr);  //将点分十进制的IP地址转换为用网络字节序表示的整数地址，失败时返回INADDR_NONE
int inet_aton(const char *strptr,struct in_Addr*inp);  //与上面的功能相同，但是将转换结果存到参数inp所指的地址结构中

char * inet_ntoa(struct in_addr in);//与上面的两个函数功能相反

```

4、创建socket

在linux中一切皆是文件

```c
#include<sys/socket.h>
#include<sys/types.h>
int socket(int domain,int type,int protocal);
//参数解释
domain:告诉系统使用哪一个底层协议，对于TCP/IP协议族，指定为：PF_INET
type:指定参数服务类型，TCP协议采用的是SOCK_STREAM（流服务），对于UDP协议，采用SOCK_UGRAM（数据报）
protocal：一般根据前两个参数就可确定，默认为0
```

socket函数成功时返回0，失败时返回-1

5、命名socket

创建socket时，我们给了地址族，但是并没有将地址族与socket进行绑定，需要进行绑定

```c
#include<sys/types.h>
#include<sys/socket.h>
int bind(int sockfd,const struct sockaddr* m_addr,socklen_t addrlen);
/*
参数解释：
bind将my_addr所指的socket地址分配给未命名的socfd文件描述符，addrlen参数指出该socket地址的长度
bind成功时返回0，失败时返回-1，并设置erron
*/
```

