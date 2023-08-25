### docker网络优化

docker容器或者Dockerfile build过程中，需要使用网络的情况，尤其是pip，有时会网络连接被拒绝，
这时候可以给docker指定dns

centos中，修改daemon.json

vi /etc/docker/daemon.json

```json
{                                                                                                                                               
  "registry-mirrors":["https://2dcmmtg1.mirror.aliyuncs.com"],                                                                                  
  "dns": ["223.5.5.5","119.29.29.29"]
}
```
