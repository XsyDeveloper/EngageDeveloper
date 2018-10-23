# 查看示例

帮助刚刚了解销售易开发者平台功能的研发人员，快速上手。Clone GitHub上的销售易developerSample样例到本地，查看业务逻辑代码开发规范。

1.首先需要在Eclipse工具上安装Git插件。具体实现步骤请参阅网上相关资料。

2.销售易提供的GitHub样例地址为：[`https://github.com/XsyDeveloper/developerSample`](https://github.com/XsyDeveloper/developerSample)

![](/assets/clonegithub.png)

3.打开Eclipse工具，Clone Github示例。

* a、打开Git Repositories页签，点击`Clone Git Repository`。
* b、填写GitHub信息，选择下载位置即可。

4.导入资源到本地，右击选择`Import Projects`，把Git项目导入本地进行查看会议室预订Demo。

```java
package other.xsy.trigger;

import java.util.Date;
import java.util.List;

import com.rkhd.platform.sdk.ScriptTrigger;
import com.rkhd.platform.sdk.exception.ScriptBusinessException;
import com.rkhd.platform.sdk.http.RkhdHttpClient;
import com.rkhd.platform.sdk.http.RkhdHttpData;
import com.rkhd.platform.sdk.log.Logger;
import com.rkhd.platform.sdk.log.LoggerFactory;
import com.rkhd.platform.sdk.model.DataModel;
import com.rkhd.platform.sdk.param.ScriptTriggerParam;
import com.rkhd.platform.sdk.param.ScriptTriggerResult;
import com.rkhd.platform.sdk.test.tool.TestTriggerTool;

import net.sf.json.JSONArray;
import net.sf.json.JSONObject;
/**
 * 检查预定会议室时会议室在选定时间内是否可用并在页面上返回提示
 * @author admin
 *
 */
public class SaveMettingTrigger implements ScriptTrigger{
	private Logger logger = LoggerFactory.getLogger();
	public static void main(String[] args) {
		//在开发环境测试Trigger的业务逻辑代码是否正确
		TestTriggerTool testTriggerTool = new TestTriggerTool();
		testTriggerTool.test("src/scriptTrigger.xml", new SaveMettingTrigger());
	}
	@Override
	public ScriptTriggerResult execute(ScriptTriggerParam scriptTriggerParam)
			throws ScriptBusinessException {
		//获取用户预定会议室的信息集合
		List<DataModel> list = scriptTriggerParam.getDataModelList();
		//获取用户所预定会议室的ID
		Object huiyishiName = list.get(0).getAttribute("MeetingRoom__c");
		//获取会议开始时间
		Object beginTime = list.get(0).getAttribute("beginTime__c");
		//获取会议结束时间
		Object end_time = list.get(0).getAttribute("endTime__c");
		String begin = beginTime.toString();
		String end = end_time.toString();
		String meetingManagerIds = huiyishiName + "";
		//判断会议开始时间和结束时间是否超过一天，不允许会议时间超过一天
		if(DateUtil.getBetweenDay(new Date(Long.parseLong(begin)), new Date(Long.parseLong(end))) > 0){
			throw new ScriptBusinessException("begin time and end time difference of more than 1 days");
		}
		//判断会议结束时间是否在开始时间之后
		if(!DateUtil.isBefore(new Date(Long.parseLong(begin)), new Date(Long.parseLong(end)))){
			throw new ScriptBusinessException("begin time must before end time ");
		}
        //查询指定时间段指定会议室的预定记录
		String firstTable = getYuding(begin,end,scriptTriggerParam.getUserId(),-1L,meetingManagerIds,0,1);
		JSONObject firstTableJson = JSONObject.fromObject(firstTable);
		JSONArray firstTableJsonArray = null;
		logger.info(firstTable);
		if(firstTableJson.get("records") != null){
			firstTableJsonArray = firstTableJson.getJSONArray("records");
		} else {
			firstTableJsonArray = new JSONArray();
		}
		if(firstTableJsonArray.size() > 0){//会议室已被预订
			throw new ScriptBusinessException("该会议室已有会议，请选择其他时间或其他会议室！");
		}
		ScriptTriggerResult scriptTriggerResult = new ScriptTriggerResult();
		scriptTriggerResult.setDataModelList(scriptTriggerParam.getDataModelList());
		return scriptTriggerResult;
	}
	/**
	 * 查询指定时间段指定会议室的预定记录
	 * @param begin
	 * @param end
	 * @param userId
	 * @param tenantId
	 * @param meetingManagerIds
	 * @param first
	 * @param size
	 * @return 指定时间段指定会议室的预定信息
	 */
	private String getYuding(String begin,String end, Long userId, Long tenantId,String meetingManagerIds,int first,int size){
		try{
			RkhdHttpClient rkhdHttpClient = new RkhdHttpClient();
			RkhdHttpData rkhdHttpData = new RkhdHttpData();
			rkhdHttpData.setCallString("/data/v1/query");
			rkhdHttpData.setCall_type("POST");
			String sql = "select id,ownerId,beginTime__c,endTime__c,MeetingRoom__c from bookMeetingRoom__c where id > 0 ";
			StringBuilder sb = new StringBuilder();
			if(meetingManagerIds != null && !meetingManagerIds.equals("")){
				 sb.append(" and MeetingRoom__c in( ").append(meetingManagerIds).append(") ");
			}
			if(begin != null && !begin.equals("") && end != null && !end.equals("")){
				sb.append(" and ((beginTime__c <= ").append(begin).append(" and endTime__c > ").append(begin).append(")");
				sb.append(" or (beginTime__c < ").append(end).append(" and endTime__c >= ").append(end).append(")");
				sb.append(" or (beginTime__c >= ").append(begin).append(" and endTime__c <= ").append(end).append(") )");
			}
			sb.append(" order by beginTime__c,endTime__c");
			sb.append(" limit ").append(first).append(",").append(size);
			sql = sql + sb.toString();
			rkhdHttpData.putFormData("q", sql);
			String s = rkhdHttpClient.performRequest(rkhdHttpData);
			return s;
		}catch(Exception e){
			logger.error("trigger异常");
			return e.getMessage();
		}
	}
	
}


```



