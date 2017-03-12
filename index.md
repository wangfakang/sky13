`nginx -s reload对性能的影响`    

一.nginx reload的过程:
=====
   首先是nginx执行这样一个命令其nginx的内部是如何做的恩?其整个过程是这样的------首先会新起一个nginxmaster进程、然后进行配置的预校验、如果配置上是没有语法错误（像端口冲突啥的是无法检测的）就会给之前的master进程发送信号、然后老的master进行配置解析、fork出新的worker进程（待新起来的进程准备工作就绪、此时master进程就会通知老的worker进程关闭相关监听套接字，即不在介绍新的请求、待老的worker上的各种事件处理完成后就退出）；
   
二.注意其实在执行nginx -s reload会有这样几个问题:   
=====
 
##  1.若有大量请求,此时会丢请求.  
   * 原因是:在某一时刻nginx的worker进程是之前的二倍,导致系统资源的下降[并发自然也下降了qps]而且多个worker之间进
行锁的竞争.      
   
## 2.若此时worker子进程比较多则会很耗费内存.    
  * 由于老的worker子进程要把当前的请求的任务处理完成才会释放worker进程的资源,所以若果老的worker和后端的机器
keepalive什么的话则在一段时间内nginx的worker进程个数会随着reload的次数成倍数增加.      

三.代码分析:
====

```c
  //ngx_reconfigure标志位为1，重新读取配置文件  
  //nginx不会让原来的worker子进程再重新读取配置文件，其策略是重新初始化ngx_cycle_t结构体，
  //用它来读取新的额配置文件再创建新的额worker子进程，销毁旧的worker子进程  
        if (ngx_reconfigure) {  
            ngx_reconfigure = 0;  
  
            //ngx_new_binary标志位为1，平滑升级Nginx  
            if (ngx_new_binary) {  
                ngx_start_worker_processes(cycle, ccf->worker_processes,  
                                           NGX_PROCESS_RESPAWN);  
                ngx_start_cache_manager_processes(cycle, 0);  
                ngx_noaccepting = 0;  
  
                continue;  
            }  
  
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reconfiguring");  
  
            //初始化ngx_cycle_t结构体  
            cycle = ngx_init_cycle(cycle);  
            if (cycle == NULL) {  
                cycle = (ngx_cycle_t *) ngx_cycle;  
                continue;  
            }  
  
            ngx_cycle = cycle;  
            ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,  
                                                   ngx_core_module);  
            //创建新的worker子进程  
            ngx_start_worker_processes(cycle, ccf->worker_processes,  
                                       NGX_PROCESS_JUST_RESPAWN);  
            ngx_start_cache_manager_processes(cycle, 1);  
  
            /* allow new processes to start */  
            ngx_msleep(100);  
  
            live = 1;  
            //向所有子进程发送QUIT信号  
            ngx_signal_worker_processes(cycle,  
                                        ngx_signal_value(NGX_SHUTDOWN_SIGNAL));  
        }  



```


欢迎一起交流学习 
====
 
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(1031379296#qq.com, 把#换成@)
* QQ: 1031379296
* weibo: [@王发康](http://weibo.com/u/2786211992/home)


Thx
====

* chunshengsterATgmail.com


Author
====
* Linux\nginx\golang\c\c++爱好者
* 欢迎一起交流  一起学习# 
* Others say good and Others good


