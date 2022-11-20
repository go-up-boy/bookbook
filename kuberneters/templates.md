# 常用指令

## 创建模板

### 1.创建Job模板
```shell
export out="--dry-run=client -o yaml"
kubectl create job template-job --image=busybox $out
```

### 2. CronJob
需要定时运行，所以要指定 --schedule="" 参数，crontab语法
```shell
export out="--dry-run=client -o yaml"
kubectl create cj template-cj --image=busybox --schedule="" $out
```

### 3. ConfigMap
```shell
export out="--dry-run=client -o yaml"        # 定义Shell变量kubectl create cm info $out
kubectl create cm info --from-literal=k=v $out
```

### Secret
```shell
export out="--dry-run=client -o yaml"
kubectl create secret generic user --from-literal=name=root $out
```

### Deployment

```shell
export out="--dry-run=client -o yaml"
kubectl create deploy ngx-dep --image=nginx $out
```


### Service
```shell
export out="--dry-run=client -o yaml"
kubectl expose deploy ngx-dep --port=80 --target-port=80 $out
```