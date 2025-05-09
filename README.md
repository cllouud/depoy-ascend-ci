# 背景
目前手动部署仓库的昇腾CI.部署到贵阳节点。

对于每个组织下的仓库，部署多个NPU卡的runner-set。
`/org1/repo1/linux-arm64-npu-1`表示`org1/repo1`仓库的1卡NPU runner set。
`linux-arm64-npu-2`代表2卡。`linux-arm64-npu-4`代表4卡。

# 部署流程
以`/org1/repo1/linux-arm64-npu-1`为例
|文件名|含义|
|--|--|
|local-storage-pvc.yaml|本地跨节点缓存|
|custom-runner-container-hook-pvc.yaml|自定义[runner-container-hook](https://github.com/actions/runner-container-hooks/)编译的`index.js`, `runner pod`调用`index.js`在同namespace下创建`container job pod`并且执行workflow命令|
|container-job-pod-template-npu-1-configmap.yaml|`container job pod`的模板|
|pre-execute-script-check-npu-configmap.yaml|由于[Bug: the device is used 8020](https://github.com/ascend-gha-runners/docs/issues/11)，配置的预检查`npu-smi info`脚本|
|runner-scale-set-values.yaml|通过`helm`安装runner set的values文件|
|runner-pod-permission.yaml|定义`runner pod`的权限，使其能够创建`container job pod`|

## 用参数填充yaml文件
比如`container-job-pod-template-npu-1-configmap.yaml`
```yaml
  namespace: <ORGANIZATION>
```
`local-storage-pvc.yaml`
```yaml
    everest.io/sfsturbo-share-id: <SECRET-SFS-ID>  # 不变
  namespace: <ORGANIZATION> # 对应每个namespace name
```

## 执行yaml文件
```bash
k apply -f local-storage-pvc.yaml
k apply -f custom-runner-container-hook-pvc.yaml
k apply -f container-job-pod-template-npu-1-configmap.yaml
k apply -f pre-execute-script-check-npu-configmap.yaml
k apply -f runner-pod-permission.yaml

# 安装runner set
NAMESPACE=<ORGANIZATION>
INSTALLATION_NAME="linux-arm64-npu-x"
helm install "${INSTALLATION_NAME}" \
    --namespace "${NAMESPACE}" \
    -f runner-scale-set-values.yaml \
    oci://ghcr.nju.edu.cn/actions/actions-runner-controller-charts/gha-runner-scale-set

# 或者更新runner set
helm upgrade -f runner-scale-set-values.yaml linux-arm64-npu-x -n "${NAMESPACE}" oci://ghcr.nju.edu.cn/actions/actions-runner-controller-charts/gha-runner-scale-set

```
