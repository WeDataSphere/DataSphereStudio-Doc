# AppConn2Linkis 工作流节点类型开发指南

> 本文主要介绍 AppConn2Linkis 工作流节点类型，以及如何开发一个 AppConn2Linkis 工作流节点。

## 一、什么是 AppConn2Linkis 工作流节点类型

很多时候，我们希望降低用户写代码的学习成本，因此想设计一个UI界面，让用户只需在表单UI中简单输入几个参数，就为用户自动生成相关代码。

因此，这种使用场景催生了 BML2Linkis 和 AppConn2Linkis 工作流节点类型。

AppConn2Linkis 与 BML2Linkis 相比，唯一不同的地方在于，BML2Linkis 工作流节点要求关联的第三方页面本身就要为用户生成可被 Linkis 某个引擎直接执行的一段代码，
并保存到 Linkis 的 PublicService 之中。

而 AppConn2Linkis 工作流节点则只要求关联的第三方页面返回一些关键信息，由在 DSS 端该工作流节点关联的 AppConn 来生成可被 Linkis 某个引擎直接执行的一段代码。

因此，请先参考 [BML2Linkis 工作流节点类型开发指南](BML2Linkis工作流节点类型开发指南.md)。

## 二、如何实现一个 AppConn2Linkis 工作流节点 

**请注意，完成 [第三方系统接入DSS开发指南](第三方系统接入DSS开发指南.md) 是本步骤开始的前提。**

您需先熟读并完成 [DSS工作流如何新增工作流节点](DSS工作流如何新增工作流节点.md) 要求的步骤之后，再按本步骤完成相关的开发。

从用户在工作流 DAG 画布上拖出一个新的 AppConn2Linkis 工作流节点开始，整个前后台交互流程与 BML2Linkis 类型唯一的区别点在于：

BML2Linkis 要求保存时必须先调用 Linkis 的 `将脚本保存到BML` 接口，将生成的代码保存到 BML。

而 AppConn2Linkis 没有这个要求，用户只需将界面的关键信息通过前端 Iframe 通信传递给 DSS 工作流即可，具体流程如下：

- 用户拖出&创建新的 AppConn2Linkis 工作流节点 
- 用户双击打开该工作流节点 
- DSS 前端调用 `getAppConnNodeUrl` 接口获取该工作流节点的 URL
- **(1)DSS 后台 调用该 AppConn 的 `RefQueryJumpUrlOperation` 获取 `jumpURL`** 
- **(2)DSS 前端通过 Iframe 打开 `jumpURL`** 
- 用户在该节点的 UI 界面编辑内容 
- **(3)用户点击保存时，通过前端 Iframe 通信将该界面的 **关键信息** 传递给 DSS 工作流** 
- DSS 工作流将 Iframe 返回的 **关键信息** 写入 DSS 工作流
- **(4)执行该工作流节点时，DSS `nodeexecution` 模块请求该 AppConn 的 `AppConn2LinkisRefExecutionOperation`，通过 _关键信息_ 获取 `AppConn2LinkisResponseRef`**
- `nodeexecution` 模块通过 `AppConn2LinkisResponseRef` 返回的 `code`、`engineType` 和 `runType` 等信息将代码提交给 Linkis 进行执行。

什么是 **关键信息**？

是指通过该 **关键信息** 能生成可被 Linkis 某个引擎执行的代码的信息。

以上流程的(1)、(2)、(3)三个步骤请参考 [BML2Linkis 工作流节点类型开发指南](BML2Linkis工作流节点类型开发指南.md) ，这里不再重复介绍。

以下重点介绍：(4)执行该工作流节点时，DSS `nodeexecution` 模块请求该 AppConn 的 `AppConn2LinkisRefExecutionOperation`，通过**关键信息**获取 `AppConn2LinkisResponseRef`

`AppConn2LinkisRefExecutionOperation` 的目的，是返回一段可被 Linkis 某个引擎执行的代码，其返回对象为 `AppConn2LinkisResponseRef`。

这里给出一个简单示例，用以说明如何通过 **关键信息** 生成代码，如下：

```java
import com.webank.wedatasphere.dss.standard.app.development.operation.AbstractDevelopmentOperation;
import com.webank.wedatasphere.dss.standard.app.development.operation.RefQueryJumpUrlOperation;
import com.webank.wedatasphere.dss.standard.app.development.ref.QueryJumpUrlResponseRef;
import com.webank.wedatasphere.dss.standard.app.development.ref.impl.OnlyDevelopmentRequestRef;
import com.webank.wedatasphere.dss.standard.app.development.utils.QueryJumpUrlConstant;
import com.webank.wedatasphere.dss.standard.common.exception.operation.ExternalOperationFailedException;
import org.apache.commons.lang3.StringUtils;
import java.util.HashMap;

public class TestAppConn2LinkisRefExecutionOperation extends AbstractDevelopmentOperation<OnlyDevelopmentRequestRef.RefJobContentRequestRefImpl, ResponseRef>
        implements AppConn2LinkisRefExecutionOperation<OnlyDevelopmentRequestRef.RefJobContentRequestRefImpl> {

    @Override
    public AppConn2LinkisResponseRef execute(OnlyDevelopmentRequestRef.RefJobContentRequestRefImpl requestRef) throws ExternalOperationFailedException {
        // myTest 为前端页面的实际 URI，请按照实际情况进行指定
        String dbName = (String) requestRef.getRefJobContent().get("dbName");
        String tableName = (String) requestRef.getRefJobContent().get("tableName");
        if(StringUtils.isBlank(dbName) || StringUtils.isBlank(tableName)) {
              AppConn2LinkisResponseRef.newBuilder().error("dbName or tableName is empty");
        }
        String code = String.format("delete from %s.%s", dbName, tableName);
        return AppConn2LinkisResponseRef.newBuilder().setCode(code).setParams(new HashMap<String, Object>())
            .setEngineType("hive").setRunType("sql").build();
    }

}
```

可以看到，该 AppConn2Linkis 工作流节点通过用户在界面输入库名和表名，协助用户通过生成对应的 HiveQL 语句删除对应的 Hive 表。

这里的 **关键信息** 就是 `dbName` 和 `tableName`，是生成删表语句的关键信息。