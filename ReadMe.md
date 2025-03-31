ComfyUI Queue Prometheus/Grafana监控方案
该方案分4步骤
1. 通过Helm安装Prometheus/Grafana
2. 获取Pod里/queue的指标，转化为Prometheus需要的格式，并暴露出来
3. 修改Prometheus的配置文件，让Prometheus可以抓取到Pod的指标
4. 配置Grafna看板

具体操作：
1. 通过Helm安装Prometheus和Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack

2. 编写python脚本实现指标的转化，并暴露到8000端口，使用prometheus_clien默认的指标路径为/metric
from prometheus_client import start_http_server, Gauge
import requests
import time
import logging

QUEUE_RUNNING = Gauge('queue_running_total', 'Number of running tasks')
QUEUE_PENDING = Gauge('queue_pending_total', 'Number of pending tasks')

def fetch_queue_metrics():
    try:
        response = requests.get("http://localhost:8848/queue")
        data = response.json()
        QUEUE_RUNNING.set(len(data["queue_running"]))
        QUEUE_PENDING.set(len(data["queue_pending"]))
    except Exception as e:
        logging.error(f"Error fetching metrics: {str(e)}")

if __name__ == '__main__':
    # 启动指标服务器（端口8000）
    start_http_server(8000)
    
    # 每10秒抓取一次数据
    while True:
        fetch_queue_metrics()
        time.sleep(10)
3. 将监控脚本添加到Dockerfile里（见下面Dockerfile内容），运行build_and_push.sh推送到ECR上
FROM nvidia/cuda:12.1.1-cudnn8-devel-ubuntu22.04

WORKDIR /app

RUN apt-get update && apt-get install -y \
    git \
    python3.10 \
    python3-pip \
    curl

RUN pip3 install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu121
RUN pip install prometheus-client requests
COPY monitor.py /app/monitor.py
RUN git clone https://github.com/comfyanonymous/ComfyUI.git
RUN cd /app/ComfyUI && git checkout v0.3.9
RUN pip3 install -r /app/ComfyUI/requirements.txt

EXPOSE 8848
CMD ["sh", "-c", "python3 /app/ComfyUI/main.py --listen 0.0.0.0 --port 8848 & python3 /app/monitor.py"]

4. 获取Prometheus的配置文件
helm get values monitoring -n default > values.yaml
5. 修改配置文件values.yaml文件，添加下面的内容到 additionalScrapeConfigs:下面，具体作用是
让 Prometheus 每隔 10 秒从 default 命名空间中查找 app 标签值包含 comfyui 的 Pod，并将这些 Pod 的地址端口号统一替换为 8000，然后从这些目标端点抓取指标数据
    additionalScrapeConfigs:
      - job_name: 'comfyui-monitoring-5' #为当前的抓取作业指定一个名称。这个名称用于在 Prometheus 中标识该作业，方便管理和监控。例如，在 Prometheus 的界面中，你可以通过这个名称来查看该作业的抓取状态和数据。
        scrape_interval: 10s #定义了 Prometheus 从目标端点抓取指标数据的时间间隔。这里设置为 10s，意味着 Prometheus 每隔 10 秒就会尝试从符合条件的目标端点抓取一次指标数据
        kubernetes_sd_configs: #配置服务发现
          - role: pod #role: pod：指定 Prometheus 要发现的 Kubernetes 资源类型为 Pod。也就是说，Prometheus 会从 Kubernetes 集群中查找符合条件的 Pod 作为抓取目标
            namespaces:
              names:
                - default #指定只在 default 命名空间中查找 Pod。这样可以缩小查找范围，只关注特定命名空间下的 Pod
        relabel_configs:  #这部分配置用于对发现的目标进行重新标记和过滤，包含两个重新标记规则
            - action: keep #表示只保留符合条件的目标，不符合条件的目标将被过滤掉
              regex: comfyui #定义一个正则表达式，用于匹配 source_labels 中的标签值。这里要求 __meta_kubernetes_pod_label_app 标签的值必须包含 comfyui
              source_labels:
                - __meta_kubernetes_pod_label_app #是 Prometheus 在发现 Kubernetes Pod 时自动添加的元数据标签，它对应 Pod 的 app 标签
            - source_labels: [__address__] #指定要处理的源标签为 __address__。__address__ 标签包含了目标的地址信息，通常格式为 ip:port
              action: replace #表示对源标签的值进行替换操作
              regex: ([^:]+)(?::\d+)? #定义一个正则表达式，用于匹配 __address__ 标签的值。这个正则表达式的含义是匹配 IP 地址部分（[^:]+），并可选地匹配端口号部分（(?::\d+)?）
              replacement: ${1}:8000 #指定替换后的内容。${1} 表示正则表达式中第一个捕获组（即 IP 地址部分），然后将端口号替换为 8000
              target_label: __address__ #指定替换后的标签名，这里仍然是 __address__。因此，这个规则的作用是将发现的目标地址的端口号统一替换为 8000
6. 更新values文件，会自动同步到Prmetheus的配置文件里
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n default -f  overwrite_values.yaml （最好改个名字，备份最原始的values文件）
7. 到Grafana里按如下方式配置看板，这里无论Pod扩展，收缩，在Dashboard上都会将label为comfyui的pod指标抓取过来
