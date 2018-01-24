* 单项查询 ：  
搜索：ABC

* 字段查询 ：  
搜索： key : value

* 通配符查询 ：  
 ？: 匹配单个字符&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例：A?C  
 \* : 匹配0个或多个字符&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例：A*C  
 注意：通配符不能放在开头 
 
 
* 范围查询  
搜索：age:[20 TO 30]&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;age:{20 TO 30}  
 注：[ ]表示端点数值包含在范围内，{ }表示端点数值不包含在范围内
 
* 逻辑操作  
AND&nbsp;&nbsp;OR&nbsp;&nbsp;&nbsp;&nbsp;例：key:value &nbsp;AND/OR &nbsp;key:value  
\+ ：搜索结果中必须包含此项  
\- ：不能含有此项  
例： +firstname:H\*&nbsp;&nbsp;-age:20&nbsp;&nbsp;city:H*    
firstname字段结果中必须存在H开头的，不能有年龄是20的，city字段H开头的可有可无  

* 分组  
(firstname:H*&nbsp;OR&nbsp;age:20)&nbsp;&nbsp;AND&nbsp;&nbsp;state:KS  
先查询名字H开头年龄或者是20的结果，然后再与国家是KS的结合

* 字段分组  
firstname:(+H* -He*)  
搜索firstname字段里H开头的结果，并且排除firstname里He开头的结果

* 转义特殊字符  
\+&nbsp;&nbsp; -&nbsp;&nbsp; &&&nbsp;&nbsp; ||&nbsp;&nbsp; !&nbsp;&nbsp; ()&nbsp;&nbsp; {}&nbsp;&nbsp; []&nbsp;&nbsp; ^&nbsp;&nbsp;"&nbsp;&nbsp; ~&nbsp;&nbsp; *&nbsp;&nbsp; ?&nbsp;&nbsp; :&nbsp;&nbsp; \