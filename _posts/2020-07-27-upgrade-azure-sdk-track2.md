---
layout: post
title:  "升级 Azure Python SDK 到 Track2 预览版本"
image: ''
date:   2020-07-27 11:30:00
tags:
- Python
- Azure
description: '此文适用于想要把 Azure Python SDK 切换到 Track2 版本的用户'
categories:
- Azure
---

## 升级 SDK
目前所有的 Track2 SDK 均处于预览版本, 用户可以从 pypi.org 上查询具体的 Service SDK 是否有 Track2 SDK, 
如 `azure-mgmt-compute` 的 Track2 预览版本: https://pypi.org/project/azure-mgmt-compute/17.0.0b1/#history

通过 pypi.org 上提示的预览版本号进行安装升级, 如安装 compute: `pip install azure-mgmt-compute==17.0.0b1`.

## 使用新的 Identity 库
在 Track2 中不再使用 `azure.common` 中的 `ServicePrincipalCredentials` 进行身份校验，
取而代之的是 [`azure-identity`](https://pypi.org/project/azure-identity/) 中的校验工具, 例如: `ClientSecretCredential`

使用 `pip install azure-identity` 安装该库，并在代码中进行如下修改:
```python
# Track1 (old)
import azure.mgmt.compute
from azure.common.credentials import ServicePrincipalCredentials

credentials = ServicePrincipalCredentials(
    client_id=client_id,
    secret=client_secret,
    tenant=tenant_id
)
compute_client = azure.mgmt.compute.ComputeManagementClient(credentials=credentials, subscription_id=self.subscription_id)

# Track2 (new)
import azure.mmgt.compute
from azure.identity import ClientSecretCredential

credential = ClientSecretCredential(
    tenant_id=tenant_id,
    client_id=client_id,
    tenant_id=tenant_id
)
compute_client = azure.mgmt.compute.ComputeManagementClient(credential=credential, subscription_id=self.subscription_id)
```

## Client 的变动
大部分的 Client API 与 Track1 一致, 只有在异步的 API 部分(RestAPI 返回 202 或者需要显式调用 `.result()` 的 API), 为了明显的与一般 API 区分, API名称前加上了`begin_`的前缀.

以创建 VM 为例, 代码的改动:
```python
# Track1 (old)
result = self.compute_client.virtual_machines.create_or_update(
    group_name,
    vm_name,
    parameters
)
result = result.result()

# Track2 (new)
result = self.compute_client.virtual_machines.begin_create_or_update(
    group_name,
    vm_name,
    parameters
)
vm = result.result()
```

完整的 Track1 与 Track2 Demo 可以参考:https://github.com/00Kai0/azure-playground/tree/master/samples/azure/createVM
