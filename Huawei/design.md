- **OpenApi**
  - 添加一条或一批record到最新的批次
  - 获取一条或一批record的标注结果
  - 新建一个批次，添加一条或一批record到新批次
  - 查询当前job中还有多少record待标注
  - 通过外部的id找到我们内部对应的id，或用我们内部的id，找到外部的id
  - webhook，record做完后通知到推送方 （所有阶段的完成都做到通知，又接收方判断这些消息的业务含义）
  - Webhook，record添加后返回appen这边的id
  - api查询当前job是否可以继续添加数据



- **对接huawei的agent**
  - 一个post接口，接收huawei的通知
  - 内部需要有个优先队列，用于可以按任务的优先级排序，用数据库做即可
  - huawei的通知到达后，先放到队列中
  - 查询到huawei的视频地址后怎么做，需要暂存到我们的消息中或下载下来存在oss
  - 要保证优先级，那么我们的系统中队列应该是空的，因为我们的队列中无法做优先级处理，我们在需要获取任务的时候，才从这边获取一个任务塞进来？





- **points**
  - external传过来数据存在oss的private下的bucket中吗，怎么展示出来
  - token的过期时间问题，若没有长期token，必须要求调用方定时登陆，最好可以增加长期token





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

  - 添加一条或一批record到最新的批次

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

  - 新建一个批次，添加一条或一批record到新批次

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
