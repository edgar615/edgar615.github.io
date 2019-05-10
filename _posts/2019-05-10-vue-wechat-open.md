---
layout: post
title: Vue微信公众号授权
date: 2019-05-10
categories:
    - Vue Wechat
comments: true
permalink: vue-wechat-open.html
---

公司有个公众号项目采用VUE开发，到部署的时候发现微信授权存在问题，网上搜了一圈，大多数是在路由上判断的，粗略看了下觉得实现太过复杂，评估一番决定使用openresty实现授权功能，抄起键盘就是干

事务的结束有两种，当事务中的所以步骤全部成功执行时，事务提交。如果其中一个步骤失败，将发生回滚操作，撤消撤消之前到事务开始时的所以操作。

将vue-router设为history模式

<pre class="line-numbers"><code class="language-javascript">
const router = new Router({
  mode:'history',
  routes
});
</code></pre>

修改nginx配置

<pre class="line-numbers"><code class="language-javascript">
location / {
   index  index.html;

   if (!-e $request_filename) {
​		rewrite ^/(.*) /index.html last;
​		break;
​	}
}
</code></pre>

增加access_by_lua_block
<pre class="line-numbers"><code class="language-javascript">
location / {
	access_by_lua_block  {
		local cjson = require "cjson"
		local requestUri = ngx.var.uri
		local appid = "xxxxxxx"
		local args, err = ngx.req.get_uri_args()
		local wechatCode = nil
		for key, val in pairs(args) do
			 if key == "code" then
				 wechatCode = val
			 end
		end
		if (wechatCode ~= nil) then
			local res = ngx.location.capture("/api/wx/access-token",
				 { method=ngx.HTTP_GET, args = { code = wechatCode} }
			 )
			 ngx.log(ngx.ERR, res.body)
			if 200 ~= res.status then
			  ngx.exit(res.status)
			end
			local resBody = cjson.decode(res.body)
			local cookieStr = "wechat-token=" .. resBody.result .. "; path=/"
			ngx.header["Set-Cookie"] = {cookieStr}						
			ngx.log(ngx.ERR, cookieStr)
		else
			-- authorize get wechat code						
			local redirectUri = "https://wx.tabaosmart.com" .. ngx.var.request_uri
			local openIdUrl = "https://open.weixin.qq.com/connect/oauth2/authorize?" 
				.. "appid=" .. appid
				.. "&redirect_uri=" .. ngx.escape_uri(redirectUri)
				.. "&response_type=code&scope=snsapi_userinfo"
				.. "&state=" .. ngx.now()
				.. "#wechat_redirect"
			ngx.redirect(openIdUrl)
		end
	}

   index  index.html;
   
   if (!-e $request_filename) {
		rewrite ^/(.*) /index.html last;
		break;
	}
}
</code></pre>

有时候还有点小问题，后面在调