## 说明

[fluent-bit](https://docs.fluentbit.io/manual/)是Linux、OSX和BSD系列操作系统的快速、轻量级日志处理器和转发器。它非常注重性能，允许从不同的源收集事件而不必复杂，可以用来替换fluend。

一般采集日志量不大的时候，fluent-bit可以直接输出到elasticsearch就可以满足要求，当采集的日志量远超过写入elasticsearch，并且达到fluent-bit缓冲极限时，部分日志会被丢弃，为了解决这个问题，先把采集的日志写入缓冲区(kafka、redis等)，然后从缓冲区读取日志再写入elasticsearch，加入消息队列这种解耦方式解决了不同服务间传输数据的速度匹配问题。这里选用redis作为缓冲队列，在满足日志采集情况下，相对于kafka更加轻量。

由于官方的fluent-bit不支持redis插件，github上有人使用go语言编写了fluent-bit的[redis插件](https://github.com/majst01/fluent-bit-go-redis-output)，可以用来采集节点和容器日志，但是用来采集kubernetes集群日志时，多级json的值出现了base64编码，不能直接查看原来的值,如下图所示：

![kibana-view](kibana-view.jpg)

<br>

具体原因是go语言的json序列化时，[]byte类型自动使用了base64编码，而不是string，解决办法是数据在序列化前把[]byte类型统一转为string，具体实现如下

```go
func bytes2string(record map[interface{}]interface{}) map[string]interface{} {
    m := make(map[string]interface{})

    for k, v := range record {
        switch t := v.(type) {
        case []byte:
			// prevent encoding to base64
            m[k.(string)] = string(t)
        case map[interface{}]interface{}:
            m[k.(string)] = bytes2string(v.(map[interface{}]interface{}))
        }
        default:
            m[k.(string)] = v
    }

    return m
}
```

<br>

## 构建镜像

> docker build -t fluent-bit-redis:latest

<br>

## 使用

### 采集容器日志

```bash
# 切换目录
cd $GOPATH/zhufuyi/example/dockerLog

# 启动服务
docker-compose up -d

# 查看服务是否启动正常
docker-compose ps

在浏览器打开kibana( http://localhost:5601 )，默认账号和密码为elastic。
```

<br>

### 采集kubernetes集群日志

```bash
# 切换目录
cd $GOPATH/zhufuyi/example/kubernetesLog

# 启动服务
kubectl apply -f logging-namespace.yml
kubectl apply -f ./logstash
kubectl apply -f ./fluent-bit

# 查看服务是否启动正常
kubectl get all -n logging

在浏览器打开kibana，查看是否采集到kubernetes集群日志。
```
