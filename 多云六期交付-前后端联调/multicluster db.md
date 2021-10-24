# multicluster 表设计

## 根证书表 hs_root_cert

| 字段名         | 类型   | 必要 | 字段描述                                   |            |
| -------------- | ------ | ---- | ------------------------------------------ | ---------- |
| gorm.Model     |        |      |                                            |            |
| root_cert_name | string | true | 根证书名<br>hs_cert_content.root_cert_name | 外键；唯一 |
| organization | string | true | 根证书所属组织名 |  |
| start_time | int | true | 证书起始时间 |  |
| end_time | int | true | 证书截止时间 |  |
| root_cert_pem  | string | true | 根证书内容 |            |
| root_key_pem   | string | true | 根证书秘钥 |            |

## mesh表 hs_mesh

| 字段名              | 类型   | 必要 | 字段描述                                                     | 约束       |
| ------------------- | ------ | ---- | ------------------------------------------------------------ | ---------- |
| gorm.Model          |        |      |                                                              |            |
| mesh_id             | string | true | mesh名称                                                     | 唯一；非空 |
| network_model       | int    | true | 网络模型<br/>0: multinetwork<br/>1: singlenetwork<br/>目前只支持multinetwork |            |
| root_cert_id        | int    | true | hs_root_cert.id                                              | 外键；唯一 |
| control_plane_model | int    | yes  | 控制面模型：<br>0: multiprimary<br>1: primaryremote          |            |

## cluster表 hs_cluster

| 字段名        | 类型   | 必要 | 字段描述                                             |            |
| ------------- | ------ | ---- | ---------------------------------------------------- | ---------- |
| gorm.Model    |        |      |                                                      |            |
| cluster_name  | string | true | 集群名                                               | 外键；唯一 |
| kubeconfig_id | string | true | hs_kubeconfig.id                                     | 外键；唯一 |
| mesh_id       | string | true | hs_mesh.mesh_id                                      | 外键；唯一 |
| network_id    | string | true | 集群所属网络名                                       |            |
| type          | string | true | 集群类型：<br>0: primary<br>1: remote<br>2: external |            |
|     host      | string |   true   | master服务器ip                                       ||
|     port      |  int   |   true   | master服务器ssh端口                                  ||
|     user      | string |   true   | root账号                                             ||
|      pwd      | string |   true   | root账号密码                                         ||

## kubeconfig表 hs_kubeconfig

| 字段名     | 类型   | 必要 | 字段描述       |      |
| ---------- | ------ | ---- | -------------- | ---- |
| gorm.Model |        |      |                |      |
| content    | string | true | kubeconfig内容 |      |

