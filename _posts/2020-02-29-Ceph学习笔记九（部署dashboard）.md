## ceph部署dashboard  
``` bash  
启动dashboard模块
#ceph mgr module enable dashboard
关闭https
#ceph config set mgr mgr/dashboard/ssl false
设置登陆地址
#ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
此处设置端口，80端口在这里设置不起作用，换成其他端口就可以
#ceph config set mgr mgr/dashboard/server_port 8000 
设置登陆用户和密码
#ceph dashboard set-login-credentials admin '******'
重启服务
#ceph mgr module disable dashboard
#ceph mgr module enable dashboard
查看服务服务配置
# ceph mgr module ls
{
    "enabled_modules": [
        "balancer",
        "crash",
        "dashboard",
        "iostat",
        "prometheus",
        "restful",
        "status"
    ],
# ceph mgr services
{
    "dashboard": "http://0.0.0.0:8000/",
    "prometheus": "http://ceph-master:9283/"
}
``` 
现在就可以用地址加端口访问了。  
