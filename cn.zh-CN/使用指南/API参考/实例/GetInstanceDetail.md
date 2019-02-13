# GetInstanceDetail {#concept_xx2_pwz_qgb .concept}

GetInstanceDetail API可以获得运行实例的DAG（有向无环图）图。

## GetInstanceDetail请求参数 {#section_cjn_dxz_qgb .section}

|参数|类型|是否必选|示例值|描述|
|instanceId|Long|是|`-1`|InstanceID，流作业只有一个运行实例，此处填`-1L`，指在线上运行的，批作业可以通过ListInstance接口或者Startjob接口获得。|
|jobName|String|是|`job1`|作业名称|
|projectName|String|是|`project1`|项目名称|

## GetInstanceDetail返回参数 {#section_cbr_fxz_qgb .section}

|参数|类型|示例值|描述|
|RequestId|String|`FD0FF8C0-779A-45EB-9674-FF3E127B10D2`|请求ID，方便foas定位问题|
|Detail|String|`{a:b}`|作业运行的DAG信息|

## GetInstanceDetail请求示例 {#section_c1g_hxz_qgb .section}

```
http(s)://[Endpoint]/?Action=undefined
&<公共请求参数>
```

