
- **Agent**

  agent由以下几部分构成：

  - Service，用于提供api给huawei调用，以及处理保存huawei的taskid、访问huawei获取资源等。
  - Scheduler，用于定时从队列中取优先度在前的一些任务，构建record并通过A9的openapi把任务送进A9的相应job中。
  - DB，用于保存huawei的task，并记录相应task的完成情况



​       agent中保存的task定义，用于数据库中表设计：

|     Feild     |                         Description                         |
| :-----------: | :---------------------------------------------------------: |
|      id       |                           主键id                            |
|  external_id  |             外部系统的id，对应huawei的recordId              |
|   record_id   |                   对应A9上面job的recordId                   |
|    job_id     |                       对应A9上的jobId                       |
|    status     | 这条任务的状态：NEW, PREPARED, SUBMITTED, LABELED, FINISHED |
|   video_url   |                     待翻译视频的oss地址                     |
| subtitle_url  |                 待翻译视频的参考字幕oss地址                 |
|  result_url   |                    翻译结果字幕的oss地址                    |
|   priority    |            任务的优先级，0-9，数字越大优先级越大            |
| created_time  |                 创建时间，即任务的接收时间                  |
| Updated_time  |                        最后更新时间                         |
|   lang_code   |                     待校准字幕文件语种                      |
| subtitle_type |                      字幕文件类型: SRT                      |



![img](https://github.com/jiecaoappen/detail-design/blob/master/Huawei/Agent.png)



- **A9的OpenAPI**

  - 添加一条或一批record到最新的批次（幂等操作）

    ```java
    - Post  /project/v2/job/record/external/records
    - RequestBody: 
      - String jobId
      - List<ExternalRecord> records
    ```

    ExternalRecord定义：

  |   Field    | Description  |
  | :--------: | :----------: |
  | externalId | 外部的任务id |
  | sourceFile | 任务文件地址 |
  |    memo    |     备注     |

  - 获取一条或一批record的标注结果

    ```java
    - Get  /project/v2/job/record/external/records
    - Request param:
      - String jobId
      - List<Integer> records  /*A9平台的recordid*/
    - Response:
      - List<RecordResult> recordResults
    ```

    RecordResult定义：

  |   Field    |   Description    |
  | :--------: | :--------------: |
  | externalId |   外部的任务id   |
  |  recordId  | A9内部的recordId |
  |   Result   |   标注结果地址   |

  - 新建一个批次，添加一条或一批record到新批次（幂等操作）

    ``` java
    - Post  /project/v2/job/record/external/records-new-batch
    - RequestBody: 
      - String jobId
      - List<ExternalRecord> records
    ```

  - 查询当前job中还有多少record待标注

    ```java
    - Get  /project/v2/job/record/external/records-umlabelled
    - Request param:
      - String jobId
    - Response:
      - Int recordNum
    ```

  - 根据外部id找到A9任务中的recordId（待定）

  - 根据A9任务中的recordId找到外部Id（待定）

  - webhook，record各阶段通知到推送方（包括添加数据，标注完成，质检完成）

    - https://success.appen.com/hc/en-us/articles/201856249-Appen-Webhook-Basics

  - 查询当前job是否可以继续添加任务（由于job的状态等限制）

    ```java
    - Get  /project/v2/job/record/external/records-allow
    - Request param:
      - String jobId
    - Response:
      - Boolean allow
    ```



- **A9的OpenApi权限解决方案**
  - 新增一个role：EXTERNAL_USER，code排在靠后位置，所有openapi需要有该role的访问权限
  - 每次有新的client需要调用openapi需要在A9注册一个账号，并赋予EXTERNAL_USER访问权限，当需要禁止该合作方访问api时，取消相应账号的EXTERNAL_USER角色
  - 外部调用方的token需要我们帮助一次生成（根据项目长度设置token过期时间），是一个长期有效的token，client不需要反复登陆去刷新token


- **A9的内部改动**

  - 针对与外部集成的任务，任务每个阶段完成后，必须触发webhook回调通知client，这种集成任务的job，其targetTopic会起一个新的consumer，该consumer会负责收到消息后回调webhook，成功后ack该消息，未成功则可以重复收到消息一直尝试。
  - taskrecord中需要增加externalId，用于标示对应的外部系统的任务id。放在source这个map中，key为externalid。
  - 增加一个mapping表，用于保存externalId和recordId的对应关系，添加record时，先查询mapping表查看是否已经存在对应的externalId，若已存在，不做添加，不存在则添加。（发送到pulsar和写入mapping表，这两者涉及2PC，可能有数据一致性问题）
