---
title: linux shell聊天室
date: 2016-09-17 17:38:43
tags: ["c","linux","网络编程"]
---


## 需求分析

里程碑0：

1、基于C/S架构的聊天室，分为客户端和服务器。
2、客户端登陆时输入服务器IP、port、 昵称不能重复

里程碑1：

3、可以群发消息或指定接收人
4、admin账号可以踢人
5、新加入用户可以看到聊天历史记录

里程碑2：

5、可以互传文件
6、保存聊天信息

里程碑3：

7、第一次登录时，需要注册。

## 技术设计

1、在shell下的图形库：curses
2、多线程：pthread
3、网络：socket
4、消息结构：

```c
struct msg_st  
{  
    char name[256];  
    char to[256];  
    char data[4096];  
    char cmd[256];  
/*
 * 其中，关于cmd内容：
 * login 登陆
 * offline 服务器即将踢人
 * user 服务发送的用户列表，数据在data里
 * chat 表示这是一条消息，数据在data里
 */
};
```

5、服务器用了epoll来监听多个fd


## 登陆界面

![登陆界面](http://img.blog.csdn.net/20141228204352953?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2hlbnl1Zmx5aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 聊天界面
![聊天界面](http://img.blog.csdn.net/20141228193822585?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2hlbnl1Zmx5aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

<!--more-->

## 源码

```c
//client.c  
#include<unistd.h>  
#include<stdio.h>  
#include<stdlib.h>  
#include<errno.h>  
#include<sys/types.h>  
#include<sys/socket.h>  
#include<netdb.h>  
#include<netinet/in.h>  
#include<arpa/inet.h>  
#include<fcntl.h>  
#include<sys/ioctl.h>  
#include<sys/select.h>  
#include<pthread.h>  
#include<ncurses.h>  
#include <sys/epoll.h>  
#include "shared_vals.h"  
#include"chat_net.h"  
#include"chat_ui.h"  
  
char promote[256]= {0};  
  
int main(int argc, char * argv[])  
{  
  
    ui_init();  
  
  
    do  
    {  
        ui_login(ip,&port,name);  
        if(0==net_login(ip,&port,name))  
        {  
            ui_alert("Login ok!");  
            break;  
        }  
        else  
        {  
            ui_alert("Login failed!");  
        }  
    }  
    while(1);  
  
    ui_init_wins();  
  
    ui_refresh_wins();  
    wprintw(win[1],"IP:%s\nPORT:%d\nNAME:%s",ip,port,name);  
    ui_refresh_wins();  
  
//    int i=0;  
//    for(i=0; i<100; i++)  
//        ui_print_msg("This is a test message!");  
  
  
  
  
    //generate promote as [from] [to]  
    sprintf(promote,"[%s]:", name);  
    //start recev process  
  
    pthread_t thread;  
    int res=pthread_create(&thread,NULL,(void *)net_getmsg,(void*)NULL);  
    if(res!=0)  
    {  
          ui_alert("Error");  
          getch();  
    }  
    while(1)  
    {  
        ui_promote(promote);  
  
        int c=0;  
        struct msg_st msg_client;  
        memset((void *)&msg_client,0,sizeof(msg_client));  
        ui_getmsg(msg_client.data);  
        strcpy(msg_client.name,name);  
        strcpy(msg_client.cmd,"chat");  
        write(sockfd,&msg_client,sizeof(msg_client));  
  
  
    }  
  
    net_end();  
    ui_end();  
  
  
    exit(0);  
}  

```


```c

//chat_net.h  
#ifndef _CHAT_NET_H_  
#define _CHAT_NET_H_  
  
#include<unistd.h>  
#include<stdio.h>  
#include<stdlib.h>  
#include<errno.h>  
#include<sys/types.h>  
#include<sys/socket.h>  
#include<netdb.h>  
#include<netinet/in.h>  
#include<arpa/inet.h>  
#include<fcntl.h>  
#include<sys/ioctl.h>  
#include<sys/select.h>  
#include<pthread.h>  
#include<ncurses.h>  
#include <sys/epoll.h>  
#include <time.h>  
#include "shared_vals.h"  
#include  "chat_ui.h"  
int sockfd;  
int len;  
struct sockaddr_in address;  
int result;  
char ip[256];  
int  port;  
char name[256];  
int net_login(char *ip,int * port,char *name)  
{  
  
  
    sockfd=socket(AF_INET,SOCK_STREAM,0);  
    address.sin_family=AF_INET;  
    address.sin_addr.s_addr=inet_addr("127.0.0.1");  
    address.sin_port=htons(*port);  
    len=sizeof(address);  
    result=connect(sockfd,(struct sockaddr *)&address,len);  
    if(result==-1)  
    {  
        printf("ERROR:%s\n",strerror(errno));  
        close(sockfd);  
        return -1;  
    }  
  
    struct msg_st msg;  
    memset((void *)&msg,0,sizeof(msg));  
    sprintf(msg.cmd,"login");  
    strcpy(msg.name,name);  
    //  fgets(msg.name,256,stdin);  
    //printf("checking name [%s]...\n",msg.name);  
    write(sockfd,& msg,sizeof( msg));  
  
    usleep(100000);  
    int nread=0;  
    ioctl(sockfd,FIONREAD,&nread);  
    if(nread!=0)  
    {  
        read(sockfd,&msg,sizeof(msg));  
        if(strcmp("ok",msg.data)==0)  
        {  
            return 0;//login ok  
        }  
        else  
        {  
  
            // printf("Login failed!\n");  
            close(sockfd);  
            return -1;//login failed  
        }  
    }  
//   printf("Login failed!\n");  
    close(sockfd);  
    return -1;//login failed  
  
}  
  
void net_getmsg()  
{  
  
    while(1)  
    {  
        usleep(100000);  
        int nread=0;  
        ioctl(sockfd,FIONREAD,&nread);  
        if(nread!=0)  
        {  
            struct msg_st msg_read;  
            read(sockfd,&msg_read,sizeof(msg_read));  
            if(strncmp(msg_read.cmd,"chat",4)==0)  
            {  
                struct tm *tm_ptr;  
                time_t the_time;  
                (void)time(&the_time);  
                tm_ptr=gmtime(&the_time);  
                char buff[256]={0};  
                sprintf(buff,"\n[%02d:%02d:%02d]",tm_ptr->tm_hour,tm_ptr->tm_min,tm_ptr->tm_sec);  
                strcat(buff,msg_read.name);  
                strcat(buff,":");  
                ui_print_msg(buff);  
                ui_print_msg(msg_read.data);  
            }  
            else if(strncmp(msg_read.cmd,"user",4)==0)  
            {  
                 ui_list_users(msg_read.data);  
                 ui_input_win_active();  
            }  
            else if(strncmp(msg_read.cmd,"chat",4)==0)  
            {  
            }  
            else if(strncmp(msg_read.cmd,"chat",4)==0)  
            {  
            }  
            else  
            {  
            }  
            //debug  
  
            //todo  
  
        }  
    }  
    pthread_exit(NULL);  
  
}  
  
void net_end()  
{  
  
    close(sockfd);  
  
}  
#endif 

```

```c
//chat_ui.h  
#ifndef _CHAT_UI_H_  
#define _CHAT_UI_H_  
#include<unistd.h>  
#include<stdio.h>  
#include<stdlib.h>  
#include<curses.h>  
#define WIN_MAX  10  
#define USR_MAX  20l  
  
//need from other files  
//char ip[256];  
//int port;  
//char name[256];  
  
//char *usr_list[USR_MAX]={"Shenyu",  
//                            "Wanglm",  
 //                           NULL};  
  
  
////  
WINDOW * win[WIN_MAX ]={0};//max open win = 10  
int cur_x,cur_y;  
  
  
  
void ui_refresh_wins() //refresh all open wins  
{  
    int i=0;  
    for(i=0;i<WIN_MAX ;i++)  
    {  
        if(win [i]!=NULL)  
        {  
            touchwin(win [i]);  
            wrefresh(win [i]);  
        }  
    }  
    //the input always active  
  
}  
  
void ui_input_win_active()  
{  
      touchwin(win [2]);  
      wrefresh(win [2]);  
}  
int ui_init()  
{  
    initscr();  
    refresh();  
  
    start_color();  
  
    init_pair(1,COLOR_BLACK,COLOR_CYAN);  
    init_pair(2,COLOR_YELLOW,COLOR_BLUE);  
    init_pair(3,COLOR_YELLOW,COLOR_RED);  
  
  
}  
int ui_init_wins()  
{  
    win[0]=newwin(20,10,0,0);  
    win[1]=newwin(20,50,0,10);  
    win[2]=newwin(1,60,20,0);  
  
  
    wbkgd(win[0],COLOR_PAIR(1));  
    wbkgd(win[1],COLOR_PAIR(2)|A_BOLD);  
    wbkgd(win[2],COLOR_PAIR(3)|A_BOLD);  
    scrollok(win[0],TRUE);  
    scrollok(win[1],TRUE);  
  
  
    keypad(win[0],TRUE);  
    keypad(win[1],TRUE);  
    keypad(win[2],TRUE);  
    keypad(stdscr, TRUE);  
}  
int ui_login(char *ip,int * port,char *name)  
{  
    win[3]=newwin(10,40,5,10);  
    wbkgd(win[3],COLOR_PAIR(3)|A_BOLD);  
  
    wrefresh(win[3]);  
    mvwprintw(win[3],3,3,"Server   IP:\n");  
    mvwprintw(win[3],4,3,"Server port:\n");  
    mvwprintw(win[3],5,3,"User name  :\n");  
    wrefresh(win[3]);  
  
    wmove(win[3],3,16);  
    wrefresh(win[3]);  
    wgetnstr(win[3],ip,100);  
  
    wmove(win[3],4,16);  
    wrefresh(win[3]);  
    wscanw(win[3],"%d", port);  
  
    wmove(win[3],5,16);  
    wrefresh(win[3]);  
    wgetnstr(win[3],name,100);  
    delwin(win[3]);  
  
}  
  
  
void ui_list_users(char * data)  
{  
int i=0;  
wclear(win[0]);  
 wprintw(win[0],"--users--\n");  
 wprintw(win[0],"%s\n",data);  
wrefresh(win[0]);  
  
}  
  
void ui_print_msg(char *msg)  
{  
  
   wprintw(win[1],"%s\n",msg);  
   wrefresh(win[1]);  
}  
  
void ui_alert(char *msg)  
{  
    win[4]=newwin(10,40,5,10);  
    wbkgd(win[4],COLOR_PAIR(3)|A_BOLD);  
    wrefresh(win[4]);  
    mvwprintw(win[4],5,3,"%s\n   Press any key to continue...",msg);  
    wrefresh(win[4]);  
    getch();  
    delwin(win[3]);  
  
  
}  
  
void ui_end()  
{  
    endwin();  
}  
void ui_promote(char *msg)  
{  
 ui_input_win_active();  
 wclear(win[2]);  
 wprintw(win[2],"%s",msg);  
 wrefresh(win[2]);  
}  
void ui_getmsg(char * buff)  
{  
 ui_input_win_active();  
 wgetnstr(win[2],buff,4096);  
  
}  
//sample programs for UI  
//int main(int argc, char * argv[])  
//{  
//  
//  
//    ui_init();  
//    ui_login(ip,&port,name);  
//  
//    ui_init_wins();  
//    ui_list_users( usr_list );  
//    ui_refresh_wins();  
//    wprintw(win[1],"IP:%s\nPORT:%d\nNAME:%s",ip,port,name);  
//    ui_refresh_wins();  
//  
//int i=0;  
//for(i=0;i<100;i++)  
//    ui_print_msg("This is a test message!");  
//  
//  
//    ui_refresh_wins();  
//    getch();  
//}  
  
#endif // _CHAT_UI_H_  

```


```c
#ifndef _SHARED_VALS_H_  
#define _SHARED_VALS_H_  
  
struct msg_st  
{  
    char name[256];//发送信息人  
    char to[256];  //接收人  
    char data[4096];//数据  
    char cmd[256];//消息的类型  
};  
  
//login 登陆  
//offline 服务器即将踢人  
//user 服务发送的用户列表，数据在data里  
//chat 表示这是一条消息，数据在data里  
  
#endif  
```

```c
#include<unistd.h>  
#include<stdio.h>  
#include<stdlib.h>  
#include<errno.h>  
#include<sys/types.h>  
#include<sys/socket.h>  
#include<netdb.h>  
#include<netinet/in.h>  
#include<arpa/inet.h>  
#include<fcntl.h>  
#include<sys/ioctl.h>  
#include<sys/select.h>  
#include<pthread.h>  
#include <sys/epoll.h>  
#define MAXEVENTS 64  
  
struct msg_st  
{  
    char name[256];  
    char to[256];  
    char data[4096];  
    char cmd[256];  
};  
#define MAX_CLIENTS 256  
int chat_clients_sets[MAX_CLIENTS]={0};  
char *chat_clients_names [MAX_CLIENTS] = {0};  
int chat_clients_count=0;  
  
  
int chat_clients_add(int fd,char * _name)  
{  
    int i=0;  
    for(i=0; i<MAX_CLIENTS; i++)  
    {  
        if(chat_clients_sets[i]==0)  
        {  
            chat_clients_sets[i]=fd;  
  
            int len=strlen(_name)+1;  
            char *ptr=malloc(len*sizeof(char));  
            memset(ptr,0,sizeof(ptr));  
            strcpy(ptr,_name);  
            *(chat_clients_names+i)=ptr;  
  
            chat_clients_count++;  
            return 0;  
        }  
    }  
    return -1;  
}  
int chat_clients_del(int fd)  
{  
    int i=0;  
    for(i=0;i<MAX_CLIENTS;i++)  
    {  
        if(chat_clients_sets[i]==fd)  
        {  
            chat_clients_sets[i]=0;  
            free(*(chat_clients_names+i));  
            *(chat_clients_names+i)=NULL;  
            chat_clients_count--;  
            return 0;  
        }  
    }  
    return -1;  
}  
int chat_user_list(struct msg_st * m)  
{  
    memset( m,0,sizeof(struct msg_st));  
    strcpy(m->cmd,"user");  
    int i=0;  
    for(i=0;i<MAX_CLIENTS;i++)  
    {  
        if(*(chat_clients_names+i)!=NULL)  
        {  
            char buff[4096];  
            memset(buff,0,sizeof(buff));  
  
            sprintf(buff,"%s\n",*(chat_clients_names+i));  
            strcat(m->data,buff);  
  
        }  
  
    }  
  
  
}  
  
int chat_clients_sendto_all(struct msg_st * mymsg)  
{  
  
    int j=0;  
    for (j = 0; j < MAX_CLIENTS; j++)  
    {  
        if(chat_clients_sets[j]!=0)  
        {  
  
            printf("send to %s\n",(*mymsg).name);  
          int    s = write (chat_clients_sets[j] , mymsg, sizeof(struct msg_st));  
            if (s == -1)  
            {  
                perror ("write");  
                abort ();  
            }  
  
        }  
  
  
  
    }  
  
}  
static int  
create_and_bind (char *port)  
{  
    struct addrinfo hints;  
    struct addrinfo *result, *rp;  
    int s, sfd;  
  
    memset (&hints, 0, sizeof (struct addrinfo));  
    hints.ai_family = AF_UNSPEC;     /* Return IPv4 and IPv6 choices */  
    hints.ai_socktype = SOCK_STREAM; /* We want a TCP socket */  
    hints.ai_flags = 0x0001;  /* All interfaces */  
  
    s = getaddrinfo (NULL, port, &hints, &result);  
    if (s != 0)  
    {  
        fprintf (stderr, "getaddrinfo: %s\n", gai_strerror (s));  
        return -1;  
    }  
  
    for (rp = result; rp != NULL; rp = rp->ai_next)  
    {  
        sfd = socket (rp->ai_family, rp->ai_socktype, rp->ai_protocol);  
        if (sfd == -1)  
            continue;  
        s = bind (sfd, rp->ai_addr, rp->ai_addrlen);  
        if (s == 0)  
        {  
            /* We managed to bind successfully! */  
            break;  
        }  
  
        close (sfd);  
    }  
    if (rp == NULL)  
    {  
        fprintf (stderr, "Could not bind\n");  
        return -1;  
    }  
    freeaddrinfo (result);  
    return sfd;  
}  
  
static int  
make_socket_non_blocking (int sfd)  
{  
    int flags, s;  
  
    flags = fcntl (sfd, F_GETFL, 0);  
    if (flags == -1)  
    {  
        perror ("fcntl");  
        return -1;  
    }  
  
    flags |= O_NONBLOCK;  
    s = fcntl (sfd, F_SETFL, flags);  
    if (s == -1)  
    {  
        perror ("fcntl");  
        return -1;  
    }  
  
    return 0;  
}  
  
    int sfd, s;  
    int efd;  
    struct epoll_event event;  
    struct epoll_event *events;  
  
void init(int argc, char * argv[])  
{  
  
  if (argc != 2)  
    {  
        fprintf (stderr, "Usage: %s [port]\n", argv[0]);  
        exit (EXIT_FAILURE);  
    }  
  
    sfd = create_and_bind (argv[1]);  
    if (sfd == -1)  
        abort ();  
  
    s = make_socket_non_blocking (sfd);  
    if (s == -1)  
        abort ();  
  
    s = listen (sfd, SOMAXCONN);  
    if (s == -1)  
    {  
        perror ("listen");  
        abort ();  
    }  
  
    efd = epoll_create1 (0);  
    if (efd == -1)  
    {  
        perror ("epoll_create");  
        abort ();  
    }  
  
    event.data.fd = sfd;  
    event.events = EPOLLIN | EPOLLET;  
    s = epoll_ctl (efd, EPOLL_CTL_ADD, sfd, &event);  
    if (s == -1)  
    {  
        perror ("epoll_ctl");  
        abort ();  
    }  
  
    /* Buffer where events are returned */  
    events = calloc (MAXEVENTS, sizeof event);  
}  
  
void server_accept_clients()  
{  
 /* We have a notification on the listening socket, which 
                   means one or more incoming connections. */  
                while (1)  
                {  
                    struct sockaddr in_addr;  
                    socklen_t in_len;  
                    int infd;  
                    char hbuf[NI_MAXHOST], sbuf[NI_MAXSERV];  
  
                    in_len = sizeof in_addr;  
                    infd = accept (sfd, &in_addr, &in_len);  
                    if (infd == -1)  
                    {  
                        if ((errno == EAGAIN) ||  
                                (errno == EWOULDBLOCK))  
                        {  
                            /* We have processed all incoming 
                               connections. */  
                            break;  
                        }  
                        else  
                        {  
                            perror ("accept");  
                            break;  
                        }  
                    }  
  
                     int s = getnameinfo (&in_addr, in_len,  
                                     hbuf, sizeof hbuf,  
                                     sbuf, sizeof sbuf,  
                                     NI_NUMERICHOST | NI_NUMERICSERV);  
                    if (s == 0)  
                    {  
                        printf("Server: accepted connection on descriptor %d "  
                               "(host=%s, port=%s)\n", infd, hbuf, sbuf);  
                    }  
  
                    /* Make the incoming socket non-blocking and add it to the 
                       list of fds to monitor. */  
                    s = make_socket_non_blocking (infd);  
                    if (s == -1)  
                        abort ();  
  
                    event.data.fd = infd;  
                    event.events = EPOLLIN | EPOLLET;  
                    s = epoll_ctl (efd, EPOLL_CTL_ADD, infd, &event);  
                    //add to clients sets  
                   // chat_clients_add(infd);  
                    if (s == -1)  
                    {  
                        perror ("epoll_ctl");  
                        abort ();  
                    }  
                }  
}  
int  
main (int argc, char *argv[])  
{  
  
    init(argc,argv);  
  
    /* The event loop */  
    while (1)  
    {  
        int n, i;  
  
        n = epoll_wait (efd, events, MAXEVENTS, -1);  
        for (i = 0; i < n; i++)  
        {  
            if ((events[i].events & EPOLLERR) ||  
                    (events[i].events & EPOLLHUP) ||  
                    (!(events[i].events & EPOLLIN)))  
            {  
                /* An error has occured on this fd, or the socket is not 
                   ready for reading (why were we notified then?) */  
                fprintf (stderr, "epoll error\n");  
                close (events[i].data.fd);  
                continue;  
            }  
  
            else if (sfd == events[i].data.fd)  
            {  
                server_accept_clients();  
                continue;  
            }  
            else  
            {  
                /* We have data on the fd waiting to be read. Read and 
                   display it. We must read whatever data is available 
                   completely, as we are running in edge-triggered mode 
                   and won't get a notification again for the same 
                   data. */  
                int done = 0;  
  
                while (1)  
                {  
                    ssize_t count;  
                    //char buf[512]= {0};  
                    struct msg_st mymsg;  
                    memset((void *)&mymsg,0,sizeof(mymsg));  
                    count = read (events[i].data.fd, (void *)&mymsg, sizeof(mymsg));  
  
                    if (count == -1)  
                    {  
                        /* If errno == EAGAIN, that means we have read all 
                           data. So go back to the main loop. */  
                        if (errno != EAGAIN)  
                        {  
                            perror ("read");  
                            done = 1;  
                        }  
                        break;  
                    }  
                    else if (count == 0)  
                    {  
                        /* End of file. The remote has closed the 
                           connection. */  
                        done = 1;  
                        break;  
                    }  
                    /* do a little work here*/  
  
                    /* Write the buffer to standard output */  
                    printf("Server: msg from client [%s]-[%s]\n",mymsg.name,mymsg.data);  
  
                    if(strncmp(mymsg.cmd,"login",5)==0)  
                    {  
                        printf("return login ok!\n");  
                        strcpy(mymsg.data,"ok");  
                        chat_clients_add(events[i].data.fd,mymsg.name);  
                          chat_clients_sendto_all(&mymsg);  
                          continue;  
                    }  
  
  
  
                    strcpy(mymsg.cmd,"chat");  
                    chat_clients_sendto_all(&mymsg);  
                    usleep(100000);  
                    chat_user_list(&mymsg);  
                    printf("==user==\n%s\n",mymsg.data);  
                    chat_clients_sendto_all(&mymsg);  
  
                }  
  
                if (done)  
                {  
                    printf ("Server: closed connection on descriptor %d\n",  
                            events[i].data.fd);  
  
                    /* Closing the descriptor will make epoll remove it 
                       from the set of descriptors which are monitored. */  
                    close (events[i].data.fd);  
                }  
            }  
        }  
    }  
  
    free (events);  
  
    close (sfd);  
  
    return EXIT_SUCCESS;  
}  

```

