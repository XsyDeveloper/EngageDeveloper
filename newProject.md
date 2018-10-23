# 创建一个应用程序

帮助已经熟悉销售易开发者平台功能的研发人员，使用Engage Plugin进行新建一个应用程序，并在系统中实现效果。

1.首先在Eclipse资源管理器，新建项目。

2.选择集成的`XSYProject`，点击下一步操作。

3.填写登录信息及项目名称信息后，创建新的项目。

4.编写应用程序业务逻辑代码。

* a、创建类文件。
* b、编写业务逻辑，调用API接口或自定义业务实体。

> **应用程序场景说明**
>
> 开发一个会议室预订日期查重应用程序。在预订会议室时，对预订同一会议室的不同时间段进行判断，如果预订时间和其他人已预订会议时间重合时，进行日期时间数据查重，并提醒预订人时间重合，请重新预订其他合理时间。

5.使用销售易Engage Plugin集成封装代码。

![](/assets/plugin.png)

* a、销售易登录`Credentials Settings`。
* b、进行业务逻辑代码包上传`Push to Server`。

6.上传完后，应用程序业务逻辑代码Demo查看。

* a、上传到业务逻辑代码效果。

![](/assets/meetingtrigger.png)

* b、会议室预订日期查重代码Demo。

```java
package other.xsy.trigger;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.List;

import com.rkhd.platform.sdk.ScriptTrigger;
import com.rkhd.platform.sdk.exception.ScriptBusinessException;
import com.rkhd.platform.sdk.http.RkhdHttpClient;
import com.rkhd.platform.sdk.http.RkhdHttpData;
import com.rkhd.platform.sdk.model.DataModel;
import com.rkhd.platform.sdk.param.ScriptTriggerParam;
import com.rkhd.platform.sdk.param.ScriptTriggerResult;

import net.sf.json.JSONArray;
import net.sf.json.JSONObject;
/**
 * 
 * 修改会议室预订信息
 * @author admin
 *
 */
public class UpdateMeetingTrigger implements ScriptTrigger{

    @Override
    public ScriptTriggerResult execute(ScriptTriggerParam scriptTriggerParam)
            throws ScriptBusinessException {
        List<DataModel> list = scriptTriggerParam.getDataModelList();
        Object huiyishiName = list.get(0).getAttribute("MeetingRoom__c");
        Object beginTime = list.get(0).getAttribute("beginTime__c");
        Object end_time = list.get(0).getAttribute("endTime__c");
        Object id = list.get(0).getAttribute("id");

        String begin = beginTime.toString();
        String end = end_time.toString();
        String meetingManagerIds = huiyishiName + "";

        if(DateUtil.getBetweenDay(new Date(Long.parseLong(begin)), new Date(Long.parseLong(end))) > 0){
            throw new ScriptBusinessException("begin time and end time difference of more than 1 days");
        }
        if(!DateUtil.isBefore(new Date(Long.parseLong(begin)), new Date(Long.parseLong(end)))){
            throw new ScriptBusinessException("begin time must before end time ");
        }

        String firstTable = getYuding(begin,end,scriptTriggerParam.getUserId(),-1L,meetingManagerIds,0,1,id.toString());

        JSONObject firstTableJson = JSONObject.fromObject(firstTable);
        JSONArray firstTableJsonArray = null;
        if(firstTableJson.get("records") != null){
            firstTableJsonArray = firstTableJson.getJSONArray("records");
        } else {
            firstTableJsonArray = new JSONArray();
        }

        if(firstTableJsonArray.size() > 0){
            throw new ScriptBusinessException("该会议室已有会议，请选择其他时间或其他会议室！");
        }
        ScriptTriggerResult scriptTriggerResult = new ScriptTriggerResult();
        scriptTriggerResult.setDataModelList(scriptTriggerParam.getDataModelList());
        return scriptTriggerResult;
    }


    private String getYuding(String begin,String end, Long userId, Long tenantId,String meetingManagerIds,int first,int size,String id){
        try{

            RkhdHttpClient rkhdHttpClient = new RkhdHttpClient();

            RkhdHttpData rkhdHttpData = new RkhdHttpData();
            rkhdHttpData.setCallString("/data/v1/query");
            rkhdHttpData.setCall_type("POST");


            String sql = "select id,ownerId,beginTime__c,endTime__c,MeetingRoom__c from bookMeetingRoom__c where id > 0 ";
            StringBuilder sb = new StringBuilder();
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm");
            if(meetingManagerIds != null && !meetingManagerIds.equals("")){
                 sb.append(" and MeetingRoom__c in( ").append(meetingManagerIds).append(") ");
            }
            if(begin != null && !begin.equals("") && end != null && !end.equals("")){

                sb.append(" and ((beginTime__c <= ").append(begin).append(" and endTime__c > ").append(begin).append(")");
                sb.append(" or (beginTime__c < ").append(end).append(" and endTime__c >= ").append(end).append(")");
                sb.append(" or (beginTime__c >= ").append(begin).append(" and endTime__c <= ").append(end).append(") )");
            }
            sb.append(" and id != ").append(id);
            sb.append(" order by beginTime__c,endTime__c");
            sb.append(" limit ").append(first).append(",").append(size);
            sql = sql + sb.toString();
            System.out.println(sql);
            rkhdHttpData.putFormData("q", sql);

            String s = rkhdHttpClient.performRequest(rkhdHttpData);
            return s;
        }catch(Exception e){
            return e.getMessage();
        }
    }

}
```

验证会议室预订日期查重应用程序，日期查重功能的展示效果。[^1]

![](/assets/meetingDate.png)

[^1]: 详细内容请参阅[帮助文档](https://crm.xiaoshouyi.com/doc/document/index.html)《销售易Eclipse Plugin开发手册》

