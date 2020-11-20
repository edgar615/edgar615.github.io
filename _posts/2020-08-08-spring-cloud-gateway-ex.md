---
layout: post
title: Spring Cloud Gateway - 异常处理
date: 2020-08-08
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-ex.html
---

网关在整个架构中的作用是：

1. 路由服务端应用的请求到后端应用。
2. (聚合)后端应用的响应转发到服务端应用。

假设网关服务总是正常的前提下：

对于第1点来说，假设后端应用不能平滑无损上线，会有一定的几率出现网关路由请求到一些后端的“僵尸节点(请求路由过去的时候，应用更好在重启或者刚好停止)”，这个时候会路由会失败抛出异常，一般情况是Connection Refuse。

对于第2点来说，假设后端应用没有正确处理异常，那么应该会把异常信息经过网关转发回到服务端应用，这种情况理论上不会出现异常。

其实还有第3点隐藏的问题，网关如果不单单承担路由的功能，还包含了鉴权、限流等功能，如果这些功能开发的时候对异常捕获没有做完善的处理甚至是逻辑本身存在BUG，有可能导致异常没有被正常捕获处理，走了默认的异常处理器`DefaultErrorWebExceptionHandler`，默认的异常处理器的处理逻辑可能并不符合我们预期的结果。

> 不管是Filter抛出的异常，还是Connection Refuse，网关都会返回500错误

```
$ curl -si -X POST -H 'content-type:application/json' -d '{"foo":"bar"}' http://localhost:8080/body
HTTP/1.1 500 Internal Server Error
Content-Type: application/json
Content-Length: 143

{"timestamp":"2020-11-20T05:15:00.532+00:00","path":"/body","status":500,"error":"Internal Server Error","message":"","requestId":"86ade4a8-2"}

```



Spring Cloud Gateway的异常处理是通过`DefaultErrorWebExceptionHandler`处理的，它通过`ErrorWebFluxAutoConfiguration `创建

跟踪源码发现`DefaultErrorWebExceptionHandler`是通过`DefaultErrorAttributes`的`getErrorAttributes`方法来生成响应内容

```
    protected Mono<ServerResponse> renderErrorResponse(ServerRequest request) {
        Map<String, Object> error = this.getErrorAttributes(request, this.getErrorAttributeOptions(request, MediaType.ALL));
        return ServerResponse.status(this.getHttpStatus(error)).contentType(MediaType.APPLICATION_JSON).body(BodyInserters.fromValue(error));
    }
    
    @Override
	@Deprecated
	public Map<String, Object> getErrorAttributes(ServerRequest request, boolean includeStackTrace) {
		Map<String, Object> errorAttributes = new LinkedHashMap<>();
		errorAttributes.put("timestamp", new Date());
		errorAttributes.put("path", request.path());
		Throwable error = getError(request);
		MergedAnnotation<ResponseStatus> responseStatusAnnotation = MergedAnnotations
				.from(error.getClass(), SearchStrategy.TYPE_HIERARCHY).get(ResponseStatus.class);
		HttpStatus errorStatus = determineHttpStatus(error, responseStatusAnnotation);
		errorAttributes.put("status", errorStatus.value());
		errorAttributes.put("error", errorStatus.getReasonPhrase());
		errorAttributes.put("message", determineMessage(error, responseStatusAnnotation));
		errorAttributes.put("requestId", request.exchange().getRequest().getId());
		handleException(errorAttributes, determineException(error), includeStackTrace);
		return errorAttributes;
	}
```

所以我们只需要实现自定义的`DefaultErrorAttributes`即可

```
@Component
public class CustomErrorAttributes extends DefaultErrorAttributes {

    @Override
    public Map<String, Object> getErrorAttributes(ServerRequest request, ErrorAttributeOptions options) {
        Throwable throwable = this.getError(request);
        MergedAnnotation<ResponseStatus> responseStatusAnnotation = MergedAnnotations
                .from(throwable.getClass(), MergedAnnotations.SearchStrategy.TYPE_HIERARCHY).get(ResponseStatus.class);
        HttpStatus errorStatus = determineHttpStatus(throwable, responseStatusAnnotation);
        Map<String, Object> map = new HashMap<>();
        map.put("code", determineErrorCode(throwable));
        map.put("status", errorStatus.value());
        return map;
    }

    private HttpStatus determineHttpStatus(Throwable error, MergedAnnotation<ResponseStatus> responseStatusAnnotation) {
        // 根据不同的异常返回不同的响应码
        if (error instanceof ResponseStatusException) {
            return ((ResponseStatusException) error).getStatus();
        }
        return responseStatusAnnotation.getValue("code", HttpStatus.class).orElse(HttpStatus.BAD_REQUEST);
    }

    int determineErrorCode(Throwable throwable) {
        // 根据不同的异常返回不同的错误码
        if (throwable instanceof ConnectException) {
            return 50001;
        }
        return 10045;
    }
}
```

