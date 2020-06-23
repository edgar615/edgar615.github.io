---
layout: post
title: Spring Boot集成Kaptcha实现简易验证码功能
date: 2020-06-13
categories:
    - Spring
comments: true
permalink: spring-kaptcha.html
---

网站上的验证码的作用是保护网站安全，一般网站都要通过验证码来防止机器大规模注册，机器暴力破解数据密码等危害。

在Java中我们可以通过Google提供的Kaptcha来实现验证码

> 如果网站的并发量较大，需要滑动验证、图片验证等验证码，自身不差钱的话可以考虑购买云服务


# 1. 添加依赖

```xml
<dependency>
  <groupId>com.github.penggle</groupId>
  <artifactId>kaptcha</artifactId>
  <version>2.3.2</version>
</dependency>
```

# 2. 设置参数

```java
@Configuration
public class KaptchaConfig {
	@Bean
	public DefaultKaptcha getDefaultKaptcha() {
		DefaultKaptcha defaultKaptcha = new DefaultKaptcha();
		Properties properties = new Properties();
		// 图片边框
		properties.setProperty("kaptcha.border", "yes");
		// 边框颜色
		properties.setProperty("kaptcha.border.color", "105,179,90");
		// 字体颜色
		properties.setProperty("kaptcha.textproducer.font.color", "red");
		// 图片宽
		properties.setProperty("kaptcha.image.width", "110");
		// 图片高
		properties.setProperty("kaptcha.image.height", "40");
		// 字体大小
		properties.setProperty("kaptcha.textproducer.font.size", "30");
		// session key
		properties.setProperty("kaptcha.session.key", "kaptcha_code");
		// 验证码长度
		properties.setProperty("kaptcha.textproducer.char.length", "4");
		// 字体
		properties.setProperty("kaptcha.textproducer.font.names", "宋体,楷体,微软雅黑");
    // 使用那些字符生成验证码
    properties.setProperty("kaptcha.textproducer.char.string", "ACDEFHKPRSTWX345679");
    // 使用哪些字体
    properties.setProperty("kaptcha.textproducer.font.names", "Arial");
    // 干扰线颜色
    properties.setProperty("kaptcha.noise.color", "black");

		Config config = new Config(properties);
		defaultKaptcha.setConfig(config);

		return defaultKaptcha;
	}
}
```

# 3. 生成验证码

```java
@RequestMapping("/kaptcha")
@Controller
public class KaptchaController {
 
    @Autowire
    DefaultKaptcha defaultKaptcha;

    @GetMapping
    public void generate(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        byte[] captchaChallengeAsJpeg = null;
        ByteArrayOutputStream jpegOutputStream = new ByteArrayOutputStream();
        try {
            // 生产验证码字符串并保存到session中
            String createText = defaultKaptcha.createText();
            request.getSession().setAttribute(Constants.KAPTCHA_SESSION_KEY, createText);
            BufferedImage challenge = defaultKaptcha.createImage(createText);
            ImageIO.write(challenge, "jpg", jpegOutputStream);
        } catch (IllegalArgumentException e) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }
        // 定义response输出类型为image/jpeg类型，使用response输出流输出图片的byte数组
        captchaChallengeAsJpeg = jpegOutputStream.toByteArray();
        response.setHeader("Cache-Control", "no-store");
        response.setHeader("Pragma", "no-cache");
        response.setDateHeader("Expires", 0);
        response.setContentType("image/jpeg");
        ServletOutputStream responseOutputStream = response.getOutputStream();
        responseOutputStream.write(captchaChallengeAsJpeg);
        responseOutputStream.flush();
        responseOutputStream.close();
    }
  
}
```

# 4. 校验验证码

```java
@RequestMapping("/verify")
@ResponseBody
public Result verifyKaptcha(@RequestParam("code") String code, HttpServletRequest request)
        throws Exception {
    //从session中读取验证码
    String scode = (String) request.getSession().getAttribute(KAPTCHA_SESSION_KEY);
    if(Strings.isNullOrEmpty(scode) || !scode.equalsIgnoreCase(code)){
      throw IncorrectKaptchaException();
    } else {
      return Result.success();
    }
}
```

# 6. redis

上面的例子验证码是存在session中的，在实际环境里我们一般会用redis存储验证码，这时在请求验证码的时候需要先获取一个令牌，才能在验证的时候与请求对应。

> 可以用一个借口将图片用base64编码传递给前端，前端使用src="data:image/jpeg;base64,{图片编码}"展示

# 7. AutoConfiguration

Spring Boot AutoConfiguration的实现很简单，暂时不写了

# 8. 参数

- kaptcha.border 	图片边框，可选值：yes , no 默认值 yes
- kaptcha.border.color 	边框颜色，可选值： r,g,b 或者 white,black,blue. 默认值 black
- kaptcha.image.width 	图片宽 	200
- kaptcha.image.height 	图片高 	50
- kaptcha.producer.impl 	图片实现类 	com.google.code.kaptcha.impl.DefaultKaptcha
- kaptcha.textproducer.impl 	文本实现类 	com.google.code.kaptcha.text.impl.DefaultTextCreator
- kaptcha.textproducer.char.string 	验证码的文本集合	abcde2345678gfynmnpwx
- kaptcha.textproducer.char.length 	验证码长度 	5
- kaptcha.textproducer.font.names 	字体 	Arial, Courier
- kaptcha.textproducer.font.size 	字体大小 	40px.
- kaptcha.textproducer.font.color 	字体颜色，可选值： r,g,b 或者 white,black,blue. 	black
- kaptcha.textproducer.char.space 	文字间隔 	2
- kaptcha.noise.impl 	干扰实现类 	com.google.code.kaptcha.impl.DefaultNoise
- kaptcha.noise.color 	干扰 颜色，可选值： r,g,b 或者 white,black,blue. 	black
- kaptcha.obscurificator.impl 	图片样式：
	- 水纹 com.google.code.kaptcha.impl.WaterRipple 默认值
	- 鱼眼 com.google.code.kaptcha.impl.FishEyeGimpy
	- 阴影 com.google.code.kaptcha.impl.ShadowGimpy
- kaptcha.background.impl 	背景实现类 	com.google.code.kaptcha.impl.DefaultBackground
- kaptcha.background.clear.from 	背景颜色渐变，开始颜色 	light grey
- kaptcha.background.clear.to 	背景颜色渐变， 结束颜色 	white
- kaptcha.word.impl 	文字渲染器 	com.google.code.kaptcha.text.impl.DefaultWordRenderer
- kaptcha.session.key 	session key 	KAPTCHA_SESSION_KEY
- kaptcha.session.date 	session date 	KAPTCHA_SESSION_DATE
