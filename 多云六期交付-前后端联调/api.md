# multicluster api文档

## 测试服务器连通性

### get(OK)

**url**

`https://your.domain.com/backend/api/multicluster/master/check`

**method**

POST

**request**

| field |  type  | required |              description               |
| :---: | :----: | :------: | :------------------------------------: |
| host  | string |   yes    |                服务器ip                |
| port  |  int   |   yes    |             服务器ssh端口              |
| user  | string |   yes    |                root账号                |
|  pwd  | string |   yes    |              root账号密码              |

```json
{
    "host":"10.20.144.84",
    "port":22,
    "user":"root",
    "pwd":"See_1024",
}
```

**return**

```json
{
    "message":"success",
}
```

## 获取当前集群master节点信息

### get(OK)

**url**

`https://your.domain.com/backend/api/multicluster/master/get`

**method**

POST

**return**

```json
{
    "message":"success",
    "result":{
        "ip":"10.20.144.84"
    }
}
```


## 证书管理

### upload（OK）

**url**

`https://your.domain.com/backend/api/multicluster/cert/upload`

**method**

POST

**request**

|    field     |  type  | required |  description   |
| :----------: | :----: | :------: | :------------: |
| rootCertName | string |   yes    |    根证书名    |
|   rootCert   | string |   yes    | 根证书文件内容 |
|   rootKey    | string |   yes    |   根证书秘钥   |

```json
{
    "rootCertName":"cert1",
    "rootCert":"rootcert", // base64
    "rootKey":"rootkey" // base64
}
```



**return**

```json
{
    "message":"success",
    "result":"upload successfully!"
}
```

### create（OK）

**url**

`https://your.domain.com/backend/api/multicluster/cert/create`

**method**

POST

**request**

|    field     |  type  | required |     description      |
| :----------: | :----: | :------: | :------------------: |
| rootCertName | string |   yes    |       根证书名       |
|  expiration  |  int   |   yes    | 根证书文件有效期(天) |
| organization | string |   yes    |     证书所属机构     |

```json
{
    "rootCertName":"cert1",
    "expiration":3651,
    "organization":"personal"
}
```



**return**

```json
{
    "message":"success"
}
```

### delete（OK）

**url**

`https://your.domain.com/backend/api/multicluster/cert/delete`

**method**

POST

**request**

| field | type  | required | description  |
| :---: | :---: | :------: | :----------: |
|  ids  | []int |   yes    | 根证书id数组 |

```json
{
    "ids":[
        1,2
    ],
}
```



**return**

```json
{
    "message":"success",
    "result":""
}
```



### list（OK）

**url**

`https://your.domain.com/backend/api/multicluster/cert/list`

**method**

POST

**request**

|    field     |  type  | required |         description          |
| :----------: | :----: | :------: | :--------------------------: |
| rootCertName | string |    no    |    根证书名(支持模糊查询)    |
| organization | string |    no    | 证书所属组织（支持模糊查询） |

```json
{
    "rootCertName":"cert1",
    "organization":"personal",
    "startTime":1626413602,
    "endTime":1941773602
}
```



**return**

```json
{
    "message":"success",
    "result":[
        {
            "id":1,
            "rootCertName":"cert1",
            "organization":"personal",
            "startTime":1625119963,
            "endTime":1625219963
        },
    ]
}
```





## mesh管理

### create（OK）

**url**

`https://your.domain.com/backend/api/multicluster/mesh/create`

**method**

POST

**request**

|       field       |  type  | required | description                                                  |
| :---------------: | :----: | :------: | :----------------------------------------------------------- |
|      meshId       | string |   yes    | mesh名称                                                     |
|   networkModel    |  int   |   yes    | 网络模型<br/>1: 多网络模式<br/>2: 单网络模式<br/>目前只支持多网络模式 |
| controlPlaneModel |  int   |   yes    | 控制面模型：<br>1: 多主模式<br>2: 主从模式                   |
|    rootCertId     |  int   |   yes    | 根证书id                                                     |

```json
{
    "meshId":"mesh1",
    "networkModel":1,
    "controlPlaneModel":1,
    "rootCertId":1
}
```



**return**

```json
{
    "message":"success",
    "result":""
}
```



### delete（OK）

**url**

`https://your.domain.com/backend/api/multicluster/mesh/delete`

**method**

POST

**request**

| field | type  | required | description  |
| :---: | :---: | :------: | :----------: |
|  ids  | []int |   yes    | mesh的id数组 |

```json
{
    "ids":[
        1,2,3
    ]
}
```



**return**

```json
{
    "message":"success",
    "result":""
}
```



### list（OK）

**url**

`https://your.domain.com/backend/api/multicluster/mesh/list`

**method**

POST

**request**

|       field       |  type  | required | description                                                  |
| :---------------: | :----: | :------: | :----------------------------------------------------------- |
|      meshId       | string |    no    | mesh名称                                                     |
|   networkModel    |  int   |    no    | 网络模型<br/>1: 多网络模式<br/>2: 单网络模式<br/>目前只支持多网络模式 |
| controlPlaneModel |  int   |    no    | 控制面模型：<br/>1: 多主模式<br/>2: 主从模式                 |

```json
{
    "meshId":"mesh1",
    "networkModel":1,
    "controlPlaneModel":1
}
```



**return**

```json
{
    "message":"success",
    "result":[
        {
            "id":1,
            "meshId":"mesh1",
            "networkModel":1,
            "rootCertName":"cert1",
            "controlPlaneModel":1
        },
    ]
}
```





## kubeconfig管理

### upload（OK）

**url**

`https://your.domain.com/backend/api/multicluster/kubeconfig/upload`

**method**

POST

**request**

|  field  |  type  | required |  description   |
| :-----: | :----: | :------: | :------------: |
| content | string |   yes    | kubeconfig内容 |

```json
{ "content":"YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci1gY2x1c3RlcjoKICAgIHNlcnZlcjogaHR1cHM6Ly8xMC4yMC4xNDQuODQ6NjQ1MwogIG5hbWU6IGRlZmF1bHQKLSBjbHVzdGVyOgogICAgY2VydGlmaWNhdGUtYXV1aG9yaXR5LWRhdGE6IExTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB1TFMwdENrMUpTVU41UkVORFFXSkRaMEYzU1VKQlowbENRVVJCVGtKbmEzRm9hMmxIT1hjd1FrRlJjMFpCUkVGV1RWSk5kMFZSV1VSV1VWRkVSWGR3Y21SWFNtd1tZMjAxYkdSSFZucE5RalJZUkZSSmVFMUVWWGhQVkVFeVRWUkJlazFzYjFoRVZFMTRUVVJWZUU1NlFUSk5WRUY2VFd4dmQwWlVSVlJOUWtWSFFURlZSUXBCZUUxTFlUTldhVnBZU25WYVdGSnNZM3BEUTBGVFNYZEVVVmxLUzI5YVNXaDJZMDVCVVVWQ1FsRkJSR2RuUlZCQlJFTkRRVkZ2UTJkblJVSkJTMXBzQ21sNE9WbG1VM1J1Y1ZGU2NsSkpWVFYyYlU1UlR6TmFXRkZXZDFSclFrd3JUalpzYURrMFFYWXdkUzgzVm5CbFJsTXZOVVZ1T1ZOUFpFZDFWVGh5TjFrS1NXOTJTV2c1Y2pONWRHaFFibVY1TDBFMVJVZG9XVVI1V25kd2IwbzViMWxWVTJoRldsWm9jRUV4TkdJM19EbHRNRFp1UXpoaGJEUnJVbFF3V1d4aWNRcFhSbUZrYjA1VGVFeEdhalJVYTFWTWRrWlZabTkzUVM5bFlqZDFaUzlDY1dKWE9XUTVabVJTWWxGRk9HRkNNV1JFUW5sMlVEWlVRVzFxVFZwNE4yMWxDbVpXT1RaTlNGbDNSbFp2TTJKWFQzRm1ZMmhpWlU5WlZETmpTbGRrVVdodFJHZHNSMEZsVG1GS1RFMDVXWEY1Y1VFeGJVWTNSMncwYUZab1kwcHRZbEFLYkZaeGFVOURSV2hLWVhkeFowRTFiRmhJTkc4elVqbEVObHBYVkdsRFVTdE9RbVozZUU1NFZVeDFWMHQxUTJ1NFdsUjNWVTFUTW1SNWJrVnFlbkY1S3dwWWFuSkplVTVLTDFkWlJDOVRWMHgwUTJsalEwRjNSVUZCWVUxcVRVTkZkMFJuV1VSV1VqQlFRVkZJTDBKQlVVUkJaMHRyVFVFNFIwRXhWV1JGZDBWQ1NpOTNVVVpOUVUxQ1FXWTRkMFJSV1VwTGIxcEphSFpqVGtGUlJVeENVVUZFWjJkRlFrRkhPRFJDTW1OblVGZzRPSEF6SzBadFFtaDFaamxKYUhoME5XUUtNRFpoTm5Rdk4wcDFNMVp1VjBWdVlWbHRLMkZCYWt4aFRYZEJWSEp5VFhFd2R6Vm1hVmxzTlVweVFsRmpVMFJGWVc1NWJIRjRUMmh3TTFkV1EwOUZZZ3B4V1Roc1dHbDBXVXRKU1d4T2NGZGthekpVY1d4dmRuWmFUa1YyZGtORFVUVlVSV2RDVjFoQ2JYTnNjMVJhWm5CcVpUZGpTa15NVkV4Qk5sTnNObE5vQ2pNMldGUnJkbmhJVWtsS1lVczFhbWxwVTFSMFpqWk1NbWhaYldWNFlraElibmt3TnpSUk1WZGpja1p3Y21wNFVEWlNZWE5NV1UxV1ltMXdiMmgwWXpnS1kxWk5VakpQVHk4eldpdDJRMUZRVjBSVlNWWnRlbmROU3pOU11ESkxNRE5oV1ZGa1RYbzFjRmR1VldsVlFVaEljMlJMT1ZONU9GbERXWE54TUhKNk9BcFNNa1V3ZFc1cFRuUmhZVUkwUlM4d11YSTJWakpETkRsc11USkdibmRNUkdwYVdGVmljR15ZT1RKeE5XdEpTREJCUkRJNFdFb3hVM2QxZHowS1xTMHRMUzFGVGtRZ1EwVlNWRWxHU1VOQlZFVXRMUzB1TFFvPQogICAgc2VydmVyOiBodHRwczovLzEwLjIwLjE1NC44NDo2NDQzCiAgbmFtZToga3ViZXJuZXRlcwpjb251ZXh1czoKLSBjb251ZXh1OgogICAgY2x1c3Rlcjoga3ViZXJuZXRlcwogICAgdXNlcjoga3ViZXJuZXRlcy1hZG1pbgogIG5hbWU6IGt1YmVybmV1ZXMtYWRtaW5Aa3ViZXJuZXRlcwpjdXJyZW51LWNvbnRleHQ6IGt1YmVybmV1ZXMtYWRtaW5Aa3ViZXJuZXRlcwpraW5kOiBDb25maWcKcHJlZmVyZW5jZXM6IHt9CnVzZXJzOgotIG5hbWU6IGt1YmVybmV1ZXMtYWRtaW4KICB1c2VyOgogICAgY2xpZW51LWNlcnRpZmljYXRlLWRhdGE6IExTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB1TFMwdENrMUpTVU11YWtORFFXUnhaMEYzU1VKQlowbEpUblV2YVd4aFNDOUlibWQzUkZGWlNrdHZXa2xvZG1OT1FWRkZURUpSUVhkR1ZFVlVUVUpGUjBFeFZVVUtRWGhOUzJFelZtbGFXRXAxV2xoU2JHTjZRV1ZHZHpCNVRWUkJNVTFVYTNkT2FrVjNUWHBLWVVaM11IbE5ha1V4VFZScmQwNXFSWGROZWxaaFRVUlJlQXBHZWtGV1FtZE9Wa1pCYjFSRWJrNDFZek5TYkdKVWNIUlpXRTR3V2xoS2VrMVNhM2RHZDFsRVZsRlJSRVY1UW5Ka1YwcHNZMjAxYkdSSFZucE1WMFpyQ21KWGJIVk5TVWxDU1dwQlRrSm5hM1ZvYTJsSE9YY3dRa1ZSUlVaQlFVOURRVkU1UVUxSlNVSkRaMHREUVZGRlFYSlpUM2hSU1VNdlJXbDVORmg1TVVVS1IxTlBTVWR4ZFVVclFYbHpSVk5EVVdsSWVsWktRMGg1VFZGMFlqaE1kSE12UzJOdVluUmpUMDk1YW1kcWNHRjNlRzh5SzJKc1MxUk5WRzlpVmtOMGJBcEtWa3BQTTJ1MlJYTnNUVkEyTUhCUFVTdG1UMjVzVlhCMlZuSTFWbGRZZUcxdmFGaHhlVEZZTmtGbmIxUXlkRzlEYUc1M1RXcGtOelZJU21KSU4zbERDbEJuVUU1SGJuaEplRFpSVUhSWVNGTkVabWN2YlZkV2RreHdZbXRNV1M5WUwxZDZiSFJ4YWtwVmNVaFRRemxFUkhwTGVuRktWVlppUkcxeVRqZEZlbE1LUTBkUFNGTmFWa3hRTkM5ek9HVlZOSFpIV1VaWlFtbFllVXhOVW14M2FXVnBZV1pKY3pOUE5WcFllR1ZRTDJkUlF6WlhOSGwyTkhKTFoyVnRWMFpXVlFvMGRrdE5NVTFoT1VwWGFYbFBjSFZWZW11SGVXNW9iazByVkVJNVNEQnpPVWxZYzJSSk1xazRUWFY2WkhjeVZFWXJXbkY2V1N1VWFqSktNREF4TjA1a1NuSjZORFZPVVVsRVFWRkJRbTk1WTNkS1ZFRlBRbWRPVmtoUk9FSkJaamhGUWtGTlEwSmhRWGRGZDFsRVZsSXdiRUpCZDNkRFoxbEpTM2RaUWtKUlZVZ1tRWGRKZDBSUldVcExiMXBKYUhaalRrRlJSVXhDVVVGRVoyZEZRa1ZHYkhoMVpWRkZZbmxqVVZwWUwyRlNiV1YzWVhGcGJrSTVabVpUU25aU2FsSllZZ3BrVUhjMWVXOW1aVWMzTms1MldVVkJjM1JaUTJFeFRrMU1hazFUWTFCWWFEQkVZM1pUVkhoNFVXTkllbFExZFhaTFVsTjBMM1Z4TVU1MGNHSTRaRUpMQ21wMlVGZHZVeXN4UmxkTldFWnVSRzFvTTJkU1pYQllOM1phVTBGT2VHSkpUa1ZvVWt1SWRHbFdTWEJVTXpsTWVtUm1kVlJGY1ZSV15FRTJTMGw1UkdZS1pGZzNSVTlDUWtNcmVFVklkRFp5U1dJeFVHRnpRV1JDVWxkbmVrRnBSV2RJVkdKNFRIQnJXVk41YldGSFNVY3lURVUyYURGNU5YVjVWR2hwVVdOb15RcFNVVkYxVVdaSWF6a3dTVFZVUjJoeFJUQmlhRlZCZHpWak5tRkJOM1pIZUhob1QyWk1TSGhZZWpseGNuRTVhVWRUU21JMlJtdDNNMGRYTmtKc2J6UnJDamRLVlRGclQxbG5ZVTB3VFhjMGQyTmtVbmxoZEVJeWFYWm9LMk5oZHpjMFZGSllWV3B2T1hsUlpWTkRkR1JUVEVKMFl6MEtMUzB1TFMxRlRrUWdRMFZTVkVsR1NVTkJWRVV1TFMwdExRbz1KICAgIGNsaWVudC1rZXktZGF1YTogTFMwdExTMUNSVWRKVGlCU1UwRWdVRkpKVmtGVVJTQkxSVmt1TFMwdExRcE5TVWxGYjNkSlFrRkJTME5CVVVWQmNsbFBlRkZKUXk5RmFYazBXSGd4UlVkVFQwbEhjWFZGSzBGNWMwVlRRMUZwU1hwV1NrTkllRTFSZEdJNFRIUnpDaTlMWTI1aWRHTlBUM2xxWjJwd1lYZDRieklyWW14TFZFMVViMkpXUTNSc1NsWktUek5yZGtWemJFMVFOakJ3VDFFclprOXViRlZ3ZGxaeU5WWlhXSGdLYlc5b1dIRjVNVmcyUVdkdlZESjBiME5vYm5kTmFtUTNOVWhLWWtnM2VVTlFaMUJPUjI1NFNYZzJVVkIwV1VoVFJHWm5MMjFYVm5aTWNHSnJURmd2V1FvdlYzcHNkSEZxU2xWeFNGTkRPVVJFZWt1NmNVcFZWbUpFYlhKT14wVjZVME5IVDBoVFdsWk1VRFF2Y3pobFZUUjJSMWxHV1VKcFdIbE1UVkpzZDJsbENtbGhaa2x6TTA4MVdsaDRaVkF2WjFGRE5sYzBlWFkwY2t1blpXMVhSbFpWTkhaTFRURk5ZVGhLVjJsNVQzQjFWWHB1UjNsdWFHNU5LMVJDT1Vnd2N6a1tTVmh6WkVreldUaE5kWHBrZHpKVVJpdGFjWHBZSzFScU1rb3dNREUzVG1SeWVqUTFUbEZKUkVGUlFVSkJiMGxDUVZGRFIwSlVORmhyUm15Rk9EY3pSUW95V215TWQwMUpWSFF6SzFKRFJtbDJUV1pRZUU5R2NEUXJhVFpPY25wU2IydEtkbmd5YW14dlFuRlVZbFpSZERsc2VUaGlZbU5uTURsc2NubHVkWE5WQ2pkQ1wwZ3pMMlJFUWtWeU9WbGxUR1F6YUhKMGIwWkxZbEZVVW15TFlXbEZTMkZYWm5OelptdFZOMWRsV25jMlluRldOVWQ2ZUc5VU56TTRiVk5LTTFJS2RWbDZkakJaWms5bFZXcE9lWGMzU1VkM19HWTRlWFJOYkVkb15FWndXbGhGYURCM1VtbG9XVlU1TTFSV2JqVjJkRzl1WjBORFYwNXFSbVpXTURWRlpRbzBTMlpPTHpWa2VWUmhPVWRvUWpZNFYya3hNV1lyWjFaWU1sTlhaRTVNV1hCdk1YQlpUekpYWlcxdWFXdDNaemN2ZDAxcmFYZDJiRWt4T1N1ME1HOVNDalpEUWpSSGJFa3JhemR1TTNvdlNVMTNhbU5YZUM5dldtRTVTMk5HT1hFMFZtUjJkek5OTmtkVFRteE5NRGhWT1dwUmRVMHpjV1V2UVVSV11VbHdNM1VLU2psVVJsWkZPRUpCYjBkQ1FVMXBkbGMwWm14RlNVdGhUQ3RqTUhoemJTOU5aV1I2VW5wU19HOU1NMUpJVHpCT2J6RktlbU5KVFM5YVJIcENTRzAyWXdwS11zVmlSVkJKVlVsbU1zRTNZVmRHZFdNNVV6RnBXazlUTTFSUFFXTnBiVUV4VURkVk5HWm5hMUIwVlhjeVZYRlVTMjluTm5KNGJDdHZiazUzUTJkcENrTnBSMFkwVWtwS19HNVVkMVpPTmxwRkszaHZUbEJaSzBkRGFtNXdUbWRhT1VGRVduVjZja1ZSTjFacFRrcHpWRXBwTTJGU1lYaHNRVzlIUWtGT11WZ1tTVEJLTkVKNVRuRXdXbEJHTm5aaVMwNHJNbE5UZGk5R1JURjVha1pRY1ZSaU5rVm5Oblp2T1ZFM2FHSmlaMGczTkU4eWNHTnZjV3RpV2xFM2RGWkJjZ3BWVFZGeVZ6TnpibFZTWVhGdldtVmlOWFI1WkZocE9VdzFVRU14U1U4ME1ubERPVWhMYVhaNE5tSXZPVkpDZWt4V1VHUkhSRFkxYUdjdlVISjNXVXB1Q215SGVURkNZVzlpV1V3MFMwOUJNRVYzTTNSRFJuVjJXRXRtWmk5MFFtNUJPV3RvYVZoblUxSkJiMGRCWlZOSlVIUTFhekpQTHpkb2FEQmpPVGRwZVhZS14wdHhVMmRyV1haeFIwNHJaVVF2VWtrd1ZqSXhiVWd2UjFGVVoxZzFaREUxV2tGV2VrZHJSMnMwVDI5aGF6WnlaREEzYVRSRlFrZElNM3A1WVhKR15ncE9UV3A1UWtsVmVWQmxTazlDZEZKbVltaHNjMVp6WldaVlIzRlhORU5MWW5oMGVFOXVTVVo1UkdNck4wTndiVkJuZUZSeVZYVTNjMGRzVTI5R1lVTnJDblphWVhaQmNXeHRkRk5rVURWWlRFeGxjMWhVV2taclEyZFpRVVE1SzJJMk4zaHVOWEZhVjJWV15rMVNlalpvV1VONFRuSkxTVnBxVmpoNmEwUkNSSEFLVXpaVVN6TmpVbFZFZEdWWVJUUlJRak4wVEdRMU1zSk1aVXBMVlc5SFNYWlZhazkxY25CTVIyNUZTMUJpUlRCclZWaElOVEpTZEN1elZrTnZNVXhUWXdwSVIwOUhURU13Y25OM1prRnRSMmx4YTJ1bUwzTlhTR3BSY1haVFoySmpabTA1VGtKTVNVcHhTMlJoT1dKUlRuZE9NSGRtTmtacGNGTktNV1pDVG1xc1NtcEtXSGM1VVV1Q1owWk5aSGhqYUhKRGNYQkJXV2h1U1VWMFN6UkpialJvYXpaSldtOXFVM1JMVlRCT1FuVkpaa15sVWxSYWNWWjRURkozYW1oUFowY1tkRUpITjJnNWNYVlJiVVZpVEdSbEx5dDNMMDl4WTBsU1MyWnpNMlY1YWtOQk4xVnNiV3QwZVd1NVRETXlMekJ5YTNOUFlVZG5kRTFEU1dkWFIzRmhNd3BLVEVGMFZXOVhhMEZLU2twS2FUUmlZbTgxTHpKU2RVeHhiWE1yYlRCU2VXbzJRVWgzTmxsRVFUSXdUa1JYTVc1UVlXUk9DaTB1TFMwdFJVNUVJRkpUUVNCUVVrbFdRVlJGSUV1RldTMHRMUzB1Q2c9PQ=="
}
```



**return**

```json
{
    "message":"success"
}
```



### take（OK）

**url**

**塔克ch**

`https://your.domain.com/backend/api/multicluster/kubeconfig/take`

**method**

POST

**request**

| field |  type  | required |              description               |
| :---: | :----: | :------: | :------------------------------------: |
| host  | string |   yes    |                服务器ip                |
| port  |  int   |   yes    |             服务器ssh端口              |
| user  | string |   yes    |                root账号                |
|  pwd  | string |   yes    |              root账号密码              |
| path  | string |    no    | kubeconfig路径<br>默认：~/.kube/config |

```json
{
    "host":"10.20.144.84",
    "port":22,
    "user":"root",
    "pwd":"See_1024",
    "path":"~/.kube/config"
}
```



**return**

```json
{
    "message":"success"
}
```





### get（OK）

**url**

`https://your.domain.com/backend/api/multicluster/kubeconfig/get`

**method**

POST

**request**

| field | type | required | description |
| :---: | :--: | :------: | :---------: |
|  id   | int  |   yes    |     id      |

```json
{
    "id":"1"
}
```



**return**

```json
{
    "message":"success",
    "result":"YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci1gY2x1c3RlcjoKICAgIHNlcnZlcjogaHR1cHM6Ly8xMC4yMC4xNDQuODQ6NjQ1MwogIG5hbWU6IGRlZmF1bHQKLSBjbHVzdGVyOgogICAgY2VydGlmaWNhdGUtYXV1aG9yaXR5LWRhdGE6IExTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB1TFMwdENrMUpTVU41UkVORFFXSkRaMEYzU1VKQlowbENRVVJCVGtKbmEzRm9hMmxIT1hjd1FrRlJjMFpCUkVGV1RWSk5kMFZSV1VSV1VWRkVSWGR3Y21SWFNtd1tZMjAxYkdSSFZucE5RalJZUkZSSmVFMUVWWGhQVkVFeVRWUkJlazFzYjFoRVZFMTRUVVJWZUU1NlFUSk5WRUY2VFd4dmQwWlVSVlJOUWtWSFFURlZSUXBCZUUxTFlUTldhVnBZU25WYVdGSnNZM3BEUTBGVFNYZEVVVmxLUzI5YVNXaDJZMDVCVVVWQ1FsRkJSR2RuUlZCQlJFTkRRVkZ2UTJkblJVSkJTMXBzQ21sNE9WbG1VM1J1Y1ZGU2NsSkpWVFYyYlU1UlR6TmFXRkZXZDFSclFrd3JUalpzYURrMFFYWXdkUzgzVm5CbFJsTXZOVVZ1T1ZOUFpFZDFWVGh5TjFrS1NXOTJTV2c1Y2pONWRHaFFibVY1TDBFMVJVZG9XVVI1V25kd2IwbzViMWxWVTJoRldsWm9jRUV4TkdJM19EbHRNRFp1UXpoaGJEUnJVbFF3V1d4aWNRcFhSbUZrYjA1VGVFeEdhalJVYTFWTWRrWlZabTkzUVM5bFlqZDFaUzlDY1dKWE9XUTVabVJTWWxGRk9HRkNNV1JFUW5sMlVEWlVRVzFxVFZwNE4yMWxDbVpXT1RaTlNGbDNSbFp2TTJKWFQzRm1ZMmhpWlU5WlZETmpTbGRrVVdodFJHZHNSMEZsVG1GS1RFMDVXWEY1Y1VFeGJVWTNSMncwYUZab1kwcHRZbEFLYkZaeGFVOURSV2hLWVhkeFowRTFiRmhJTkc4elVqbEVObHBYVkdsRFVTdE9RbVozZUU1NFZVeDFWMHQxUTJ1NFdsUjNWVTFUTW1SNWJrVnFlbkY1S3dwWWFuSkplVTVLTDFkWlJDOVRWMHgwUTJsalEwRjNSVUZCWVUxcVRVTkZkMFJuV1VSV1VqQlFRVkZJTDBKQlVVUkJaMHRyVFVFNFIwRXhWV1JGZDBWQ1NpOTNVVVpOUVUxQ1FXWTRkMFJSV1VwTGIxcEphSFpqVGtGUlJVeENVVUZFWjJkRlFrRkhPRFJDTW1OblVGZzRPSEF6SzBadFFtaDFaamxKYUhoME5XUUtNRFpoTm5Rdk4wcDFNMVp1VjBWdVlWbHRLMkZCYWt4aFRYZEJWSEp5VFhFd2R6Vm1hVmxzTlVweVFsRmpVMFJGWVc1NWJIRjRUMmh3TTFkV1EwOUZZZ3B4V1Roc1dHbDBXVXRKU1d4T2NGZGthekpVY1d4dmRuWmFUa1YyZGtORFVUVlVSV2RDVjFoQ2JYTnNjMVJhWm5CcVpUZGpTa15NVkV4Qk5sTnNObE5vQ2pNMldGUnJkbmhJVWtsS1lVczFhbWxwVTFSMFpqWk1NbWhaYldWNFlraElibmt3TnpSUk1WZGpja1p3Y21wNFVEWlNZWE5NV1UxV1ltMXdiMmgwWXpnS1kxWk5VakpQVHk4eldpdDJRMUZRVjBSVlNWWnRlbmROU3pOU11ESkxNRE5oV1ZGa1RYbzFjRmR1VldsVlFVaEljMlJMT1ZONU9GbERXWE54TUhKNk9BcFNNa1V3ZFc1cFRuUmhZVUkwUlM4d11YSTJWakpETkRsc11USkdibmRNUkdwYVdGVmljR15ZT1RKeE5XdEpTREJCUkRJNFdFb3hVM2QxZHowS1xTMHRMUzFGVGtRZ1EwVlNWRWxHU1VOQlZFVXRMUzB1TFFvPQogICAgc2VydmVyOiBodHRwczovLzEwLjIwLjE1NC44NDo2NDQzCiAgbmFtZToga3ViZXJuZXRlcwpjb251ZXh1czoKLSBjb251ZXh1OgogICAgY2x1c3Rlcjoga3ViZXJuZXRlcwogICAgdXNlcjoga3ViZXJuZXRlcy1hZG1pbgogIG5hbWU6IGt1YmVybmV1ZXMtYWRtaW5Aa3ViZXJuZXRlcwpjdXJyZW51LWNvbnRleHQ6IGt1YmVybmV1ZXMtYWRtaW5Aa3ViZXJuZXRlcwpraW5kOiBDb25maWcKcHJlZmVyZW5jZXM6IHt9CnVzZXJzOgotIG5hbWU6IGt1YmVybmV1ZXMtYWRtaW4KICB1c2VyOgogICAgY2xpZW51LWNlcnRpZmljYXRlLWRhdGE6IExTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB1TFMwdENrMUpTVU11YWtORFFXUnhaMEYzU1VKQlowbEpUblV2YVd4aFNDOUlibWQzUkZGWlNrdHZXa2xvZG1OT1FWRkZURUpSUVhkR1ZFVlVUVUpGUjBFeFZVVUtRWGhOUzJFelZtbGFXRXAxV2xoU2JHTjZRV1ZHZHpCNVRWUkJNVTFVYTNkT2FrVjNUWHBLWVVaM11IbE5ha1V4VFZScmQwNXFSWGROZWxaaFRVUlJlQXBHZWtGV1FtZE9Wa1pCYjFSRWJrNDFZek5TYkdKVWNIUlpXRTR3V2xoS2VrMVNhM2RHZDFsRVZsRlJSRVY1UW5Ka1YwcHNZMjAxYkdSSFZucE1WMFpyQ21KWGJIVk5TVWxDU1dwQlRrSm5hM1ZvYTJsSE9YY3dRa1ZSUlVaQlFVOURRVkU1UVUxSlNVSkRaMHREUVZGRlFYSlpUM2hSU1VNdlJXbDVORmg1TVVVS1IxTlBTVWR4ZFVVclFYbHpSVk5EVVdsSWVsWktRMGg1VFZGMFlqaE1kSE12UzJOdVluUmpUMDk1YW1kcWNHRjNlRzh5SzJKc1MxUk5WRzlpVmtOMGJBcEtWa3BQTTJ1MlJYTnNUVkEyTUhCUFVTdG1UMjVzVlhCMlZuSTFWbGRZZUcxdmFGaHhlVEZZTmtGbmIxUXlkRzlEYUc1M1RXcGtOelZJU21KSU4zbERDbEJuVUU1SGJuaEplRFpSVUhSWVNGTkVabWN2YlZkV2RreHdZbXRNV1M5WUwxZDZiSFJ4YWtwVmNVaFRRemxFUkhwTGVuRktWVlppUkcxeVRqZEZlbE1LUTBkUFNGTmFWa3hRTkM5ek9HVlZOSFpIV1VaWlFtbFllVXhOVW14M2FXVnBZV1pKY3pOUE5WcFllR1ZRTDJkUlF6WlhOSGwyTkhKTFoyVnRWMFpXVlFvMGRrdE5NVTFoT1VwWGFYbFBjSFZWZW11SGVXNW9iazByVkVJNVNEQnpPVWxZYzJSSk1xazRUWFY2WkhjeVZFWXJXbkY2V1N1VWFqSktNREF4TjA1a1NuSjZORFZPVVVsRVFWRkJRbTk1WTNkS1ZFRlBRbWRPVmtoUk9FSkJaamhGUWtGTlEwSmhRWGRGZDFsRVZsSXdiRUpCZDNkRFoxbEpTM2RaUWtKUlZVZ1tRWGRKZDBSUldVcExiMXBKYUhaalRrRlJSVXhDVVVGRVoyZEZRa1ZHYkhoMVpWRkZZbmxqVVZwWUwyRlNiV1YzWVhGcGJrSTVabVpUU25aU2FsSllZZ3BrVUhjMWVXOW1aVWMzTms1MldVVkJjM1JaUTJFeFRrMU1hazFUWTFCWWFEQkVZM1pUVkhoNFVXTkllbFExZFhaTFVsTjBMM1Z4TVU1MGNHSTRaRUpMQ21wMlVGZHZVeXN4UmxkTldFWnVSRzFvTTJkU1pYQllOM1phVTBGT2VHSkpUa1ZvVWt1SWRHbFdTWEJVTXpsTWVtUm1kVlJGY1ZSV15FRTJTMGw1UkdZS1pGZzNSVTlDUWtNcmVFVklkRFp5U1dJeFVHRnpRV1JDVWxkbmVrRnBSV2RJVkdKNFRIQnJXVk41YldGSFNVY3lURVUyYURGNU5YVjVWR2hwVVdOb15RcFNVVkYxVVdaSWF6a3dTVFZVUjJoeFJUQmlhRlZCZHpWak5tRkJOM1pIZUhob1QyWk1TSGhZZWpseGNuRTVhVWRUU21JMlJtdDNNMGRYTmtKc2J6UnJDamRLVlRGclQxbG5ZVTB3VFhjMGQyTmtVbmxoZEVJeWFYWm9LMk5oZHpjMFZGSllWV3B2T1hsUlpWTkRkR1JUVEVKMFl6MEtMUzB1TFMxRlRrUWdRMFZTVkVsR1NVTkJWRVV1TFMwdExRbz1KICAgIGNsaWVudC1rZXktZGF1YTogTFMwdExTMUNSVWRKVGlCU1UwRWdVRkpKVmtGVVJTQkxSVmt1TFMwdExRcE5TVWxGYjNkSlFrRkJTME5CVVVWQmNsbFBlRkZKUXk5RmFYazBXSGd4UlVkVFQwbEhjWFZGSzBGNWMwVlRRMUZwU1hwV1NrTkllRTFSZEdJNFRIUnpDaTlMWTI1aWRHTlBUM2xxWjJwd1lYZDRieklyWW14TFZFMVViMkpXUTNSc1NsWktUek5yZGtWemJFMVFOakJ3VDFFclprOXViRlZ3ZGxaeU5WWlhXSGdLYlc5b1dIRjVNVmcyUVdkdlZESjBiME5vYm5kTmFtUTNOVWhLWWtnM2VVTlFaMUJPUjI1NFNYZzJVVkIwV1VoVFJHWm5MMjFYVm5aTWNHSnJURmd2V1FvdlYzcHNkSEZxU2xWeFNGTkRPVVJFZWt1NmNVcFZWbUpFYlhKT14wVjZVME5IVDBoVFdsWk1VRFF2Y3pobFZUUjJSMWxHV1VKcFdIbE1UVkpzZDJsbENtbGhaa2x6TTA4MVdsaDRaVkF2WjFGRE5sYzBlWFkwY2t1blpXMVhSbFpWTkhaTFRURk5ZVGhLVjJsNVQzQjFWWHB1UjNsdWFHNU5LMVJDT1Vnd2N6a1tTVmh6WkVreldUaE5kWHBrZHpKVVJpdGFjWHBZSzFScU1rb3dNREUzVG1SeWVqUTFUbEZKUkVGUlFVSkJiMGxDUVZGRFIwSlVORmhyUm15Rk9EY3pSUW95V215TWQwMUpWSFF6SzFKRFJtbDJUV1pRZUU5R2NEUXJhVFpPY25wU2IydEtkbmd5YW14dlFuRlVZbFpSZERsc2VUaGlZbU5uTURsc2NubHVkWE5WQ2pkQ1wwZ3pMMlJFUWtWeU9WbGxUR1F6YUhKMGIwWkxZbEZVVW15TFlXbEZTMkZYWm5OelptdFZOMWRsV25jMlluRldOVWQ2ZUc5VU56TTRiVk5LTTFJS2RWbDZkakJaWms5bFZXcE9lWGMzU1VkM19HWTRlWFJOYkVkb15FWndXbGhGYURCM1VtbG9XVlU1TTFSV2JqVjJkRzl1WjBORFYwNXFSbVpXTURWRlpRbzBTMlpPTHpWa2VWUmhPVWRvUWpZNFYya3hNV1lyWjFaWU1sTlhaRTVNV1hCdk1YQlpUekpYWlcxdWFXdDNaemN2ZDAxcmFYZDJiRWt4T1N1ME1HOVNDalpEUWpSSGJFa3JhemR1TTNvdlNVMTNhbU5YZUM5dldtRTVTMk5HT1hFMFZtUjJkek5OTmtkVFRteE5NRGhWT1dwUmRVMHpjV1V2UVVSV11VbHdNM1VLU2psVVJsWkZPRUpCYjBkQ1FVMXBkbGMwWm14RlNVdGhUQ3RqTUhoemJTOU5aV1I2VW5wU19HOU1NMUpJVHpCT2J6RktlbU5KVFM5YVJIcENTRzAyWXdwS11zVmlSVkJKVlVsbU1zRTNZVmRHZFdNNVV6RnBXazlUTTFSUFFXTnBiVUV4VURkVk5HWm5hMUIwVlhjeVZYRlVTMjluTm5KNGJDdHZiazUzUTJkcENrTnBSMFkwVWtwS19HNVVkMVpPTmxwRkszaHZUbEJaSzBkRGFtNXdUbWRhT1VGRVduVjZja1ZSTjFacFRrcHpWRXBwTTJGU1lYaHNRVzlIUWtGT11WZ1tTVEJLTkVKNVRuRXdXbEJHTm5aaVMwNHJNbE5UZGk5R1JURjVha1pRY1ZSaU5rVm5Oblp2T1ZFM2FHSmlaMGczTkU4eWNHTnZjV3RpV2xFM2RGWkJjZ3BWVFZGeVZ6TnpibFZTWVhGdldtVmlOWFI1WkZocE9VdzFVRU14U1U4ME1ubERPVWhMYVhaNE5tSXZPVkpDZWt4V1VHUkhSRFkxYUdjdlVISjNXVXB1Q215SGVURkNZVzlpV1V3MFMwOUJNRVYzTTNSRFJuVjJXRXRtWmk5MFFtNUJPV3RvYVZoblUxSkJiMGRCWlZOSlVIUTFhekpQTHpkb2FEQmpPVGRwZVhZS14wdHhVMmRyV1haeFIwNHJaVVF2VWtrd1ZqSXhiVWd2UjFGVVoxZzFaREUxV2tGV2VrZHJSMnMwVDI5aGF6WnlaREEzYVRSRlFrZElNM3A1WVhKR15ncE9UV3A1UWtsVmVWQmxTazlDZEZKbVltaHNjMVp6WldaVlIzRlhORU5MWW5oMGVFOXVTVVo1UkdNck4wTndiVkJuZUZSeVZYVTNjMGRzVTI5R1lVTnJDblphWVhaQmNXeHRkRk5rVURWWlRFeGxjMWhVV2taclEyZFpRVVE1SzJJMk4zaHVOWEZhVjJWV15rMVNlalpvV1VONFRuSkxTVnBxVmpoNmEwUkNSSEFLVXpaVVN6TmpVbFZFZEdWWVJUUlJRak4wVEdRMU1zSk1aVXBMVlc5SFNYWlZhazkxY25CTVIyNUZTMUJpUlRCclZWaElOVEpTZEN1elZrTnZNVXhUWXdwSVIwOUhURU13Y25OM1prRnRSMmx4YTJ1bUwzTlhTR3BSY1haVFoySmpabTA1VGtKTVNVcHhTMlJoT1dKUlRuZE9NSGRtTmtacGNGTktNV1pDVG1xc1NtcEtXSGM1VVV1Q1owWk5aSGhqYUhKRGNYQkJXV2h1U1VWMFN6UkpialJvYXpaSldtOXFVM1JMVlRCT1FuVkpaa15sVWxSYWNWWjRURkozYW1oUFowY1tkRUpITjJnNWNYVlJiVVZpVEdSbEx5dDNMMDl4WTBsU1MyWnpNMlY1YWtOQk4xVnNiV3QwZVd1NVRETXlMekJ5YTNOUFlVZG5kRTFEU1dkWFIzRmhNd3BLVEVGMFZXOVhhMEZLU2twS2FUUmlZbTgxTHpKU2RVeHhiWE1yYlRCU2VXbzJRVWgzTmxsRVFUSXdUa1JYTVc1UVlXUk9DaTB1TFMwdFJVNUVJRkpUUVNCUVVrbFdRVlJGSUV1RldTMHRMUzB1Q2c9PQ=="
}
```





## cluster管理



### check（OK）

**url**

`https://your.domain.com/backend/api/multicluster/cluster/check`

**method**

POST

**request**

|    field     |  type  | required | description                                          |
| :----------: | :----: | :------: | :--------------------------------------------------- |
|     host     | string |   yes    | master服务器ip                                       |
|     port     |  int   |   yes    | master服务器ssh端口                                  |
|     user     | string |   yes    | root账号                                             |
|     pwd      | string |   yes    | root账号密码                                         |

```json
{
    "host":"10.20.144.84",
    "port":22,
    "user":"root",
    "pwd":"See_1024"
}
```



**return**

```json
{
    "message":"success",
    "result":true
}
```



### takeover(OK)

**url**

`https://your.domain.com/backend/api/multicluster/cluster/takeover`

**method**

POST

**request**

|    field     |  type  | required | description                                         |
| :----------: | :----: | :------: | :-------------------------------------------------- |
| ifUninstall  |  bool  |   yes    | 是否需要卸载<br>- true<br>- false                   |
| clusterName  | string |   yes    | 集群名                                              |
|    meshId    | string |   yes    | mesh名                                              |
| kubeconfigId |  int   |   yes    | kubeconfig的id                                      |
|   roleType   |  int   |   yes    | 角色类型：<br>1: 主集群<br>2: 从集群<br>3: 外部集群 |
|  networkId   | string |   yes    | 自定义网络名称                                      |
|     hub      | string |   yes    | 指定镜像仓库名                                      |
|     host     | string |   yes    | master服务器ip                                      |
|     port     |  int   |   yes    | master服务器ssh端口                                 |
|     user     | string |   yes    | root账号                                            |
|     pwd      | string |   yes    | root账号密码                                        |



```json
{
    "ifUninstall":false,
    "clusterName":"cluster1",
    "meshId":"mesh1",
    "kubeconfigId":1,
    "roleType":1,
    "networkId":"network1",
    "hub":"registry.hundsun.com/hcs",
    "host":"10.20.144.84",
    "port":22,
    "user":"root",
    "pwd":"See_1024"
}
```



**return**

```json
{
    "message":"success",
    "result":{
        "log":"loginfo"
    }
}
```



### remove(OK)

**url**

`https://your.domain.com/backend/api/multicluster/cluster/remove`

**method**

POST

**request**

| field | type  | required |  description   |
| :---: | :---: | :------: | :------------: |
|  ids  | []int |   yes    | cluster id数组 |

```json
{
    "ids":[
        1
    ]
}
```



**return**

```json
{
    "message":"success"
}
```



### list(OK)

**url**

`https://your.domain.com/backend/api/multicluster/cluster/list`

**method**

POST

**request**

|    field    |  type  | required | description                                            |
| :---------: | :----: | :------: | :----------------------------------------------------- |
| clusterName | string |    no    | cluster名                                              |
|   meshId    | string |    no    | mesh名                                                 |
|  roleType   |  int   |    no    | 角色类型：<br/>1: 主集群<br/>2: 从集群<br/>3: 外部集群 |

```json
{
    "clusterName":"cluster1",
    "meshId":"mesh1",
    "roleType":1,
    "rootCertName":"cert1"
}
```



**return**

```json
{
    "message":"success",
    "reslut":[
        {
            "id":1,
            "clusterName":"cluster1",
            "meshId":"mesh1",
    		"kubeconfigId":1,
    		"roleType":1,
    		"networkId":"network1",
            "rootCertName":"cert1",
            "organization":"personal",
            "startTime":1625119963,
            "endTime":1625219963
        },
    ]
}
```

## 预测试

### deploy

**url**

`https://your.domain.com/backend/api/multicluster/pre-test/deploy`

**method**

POST

**request**

| field |  type  | required |  description   |
| :---: | :----: | :------: | :------------: |
| kubeconfigId | int | yes | kubeconfig的id |
| hub | string | yes | 指定镜像仓库名 |
|     host      | string |   yes    | master服务器ip                                         |
|     port      |  int   |   yes    | master服务器ssh端口                                    |
|     user      | string |   yes    | root账号                                               |
|      pwd      | string |   yes    | root账号密码                                           |

```json
{
    "ip":"10.20.144.84"
}
```



**return**

```json
{
    "message":"success"
}
```

### test

**url**

`https://your.domain.com/backend/api/multicluster/pre-test/test`

**method**

POST

**request**

|   field   | type | required | description |
| :-------: | :--: | :------: | :---------: |
| clusterId | int  |   yes    | cluster的id |


```json
{
    "clusterId":1
}
```



**return**

```json
{
    "message": "success",
    "result": {
        "pass": [
            {
                "req": "10.20.144.83",
                "resp": "10.20.144.165"
            },
            {
                "req": "10.20.144.83",
                "resp": "10.20.144.83"
            },
            {
                "req": "10.20.144.165",
                "resp": "10.20.144.83"
            },
            {
                "req": "10.20.144.165",
                "resp": "10.20.144.165"
            }
        ],
        "fail": null
    }
}
```

