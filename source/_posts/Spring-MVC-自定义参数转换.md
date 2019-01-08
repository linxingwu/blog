---
title: Spring MVC 自定义参数转换
date: 2019-01-08 14:12:15
tags:
 - spring
 - SpringMVC
---

碰到这么一个情况，前端传递参数，后台用一个Map来接受，我希望参数是数字的，接收到的还是数字。然而后台controller里注入到的，一定都是String。当然如果需要Integer的自己取出来转换再放回去就好了。只是不免不够优雅有点low。那么我们怎么办呢？我首先想到的就是Initbinder

在controller中添加一个方法，必须返回值为void，WebDataBinder。最常见的一种是自定义的时间转换。
```java
    @InitBinder
    public void initBinder(WebDataBinder binder) throws Exception {
        binder.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
    }

```

因为我是在开发公司项目的时候遇到的问题，最终解决之后才能写这一篇博客，所以解决过程中的代码已经没有了，现在都是随手说明一下而已。

CustomDateEditor，以及一些其他的Editor是JDK的PropertyEditor的接口的实现。这个接口本来是用于支持GUI变成的，jdk自己定义了一个简单实现PropertyEditorSUpport，然后spring继承了这个类，实现的各种自定义转换器。

这种方法有一个缺陷，就是只能在一个controller中生效。

于此类似的方法还有一个Converter，这是spring 3 中新出现的类，只有一个convert方法，有两个泛型，一个输入类型一个输出类型。convert方法中实现具体的转换逻辑。

然后经过测试只有参数Map仍然是String。因为CustomMapEditor并不是把前端参数转换成map用的，注意他的构造方法，需要传入一个Map的class，它实际作用是将一个map转换成另一个map。

initbinder和converter都失败了我不得不去看一下springmvc到底是怎么处理前端参数的。于是先从DispatcherServlet.doservice开始，一路沿着dodispatch，看到spring是如何找到handler，再调用handerAdapter的handle方法进行真正的请求处理。这个方法又是调用了RequestMappingHandlerAdapter的handleInternal方法，这里调用了invokeHandlerMethod,看名字应该是通过反射调用了Controller里的具体处理方法。

这个方法的灵魂又是invocableMethod.invokeAndHandle方法，进去再跳invokeForRequest,再跳getMethodArgumentValues.勉强算是找到了参数解析的入口了。
```java

			if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
					}
					throw ex;
				}
			}
```
这一段是判断类中保存的参数解析器集合是否能够解析出这个方法参数，如果可以，尝试调用解析器去解析一波。argumentResolvers看起来是一个HandlerMethodArgumentResolverComposite而不是一个集合，其实HandlerMethodArgumentResolverComposite就是一个纸老虎，本质还是维护了一个resolver的集合，需要判断或者解析的时候就一个个遍历一波
```java
	private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		if (result == null) {
			for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
				if (logger.isTraceEnabled()) {
					logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
							parameter.getGenericParameterType() + "]");
				}
				if (methodArgumentResolver.supportsParameter(parameter)) {
					result = methodArgumentResolver;
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```
可以看到这里加了一个缓存，找到参数类型对应的解析器了就直接缓存起来下次好直接取出来。

所以map问题的解决办法看起来已经很清晰了，在argumentResolvers的集合里把我们自定义的玩意加进去就好了。没错，看看这玩意在哪里初始化的。在RequestMappingHandlerAdapter的afterPropertiesSet中设置的
```java
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();

		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
```

可以看到，先后添加了3类参数解析器，首先是默认的参数解析器，然后initbinder的解析器，最后是返回值的处理器。所以我们关心的就是
```java
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}
```
看到了这么多解析器还真是让人激动，其中第二个RequestParamMapMethodArgumentResolver尤其让人激动，这个是@RequestParam注解的Map参数解析器。正是我所关心的情况。踏破铁鞋无觅处啊。在这里我走了一些弯路，我首先想着定义一个解析器继承它，然后回到前面寻找解析器的方法里getArgumentResolver，它是遍历找到一个就break，而我自定的解析器排在默认解析器后面。悲剧啊，永远找不到我的解析器。

怎么办呢？我又走了一些弯路，我尝试写一个自定义的注解，替换掉@RequestParam，这样在RequestParamMapMethodArgumentResolver判断是够支持对应参数时就会返回不支持，然后轮到我自定义的子类，返回支持，不就美滋滋？不料还是不行，因为还有一个MapMethodProcessor排在我前面，这个就无解了，它判断是否支持参数标准是参数是不是map，注解好写，map不好重新实现啊。当初选了map做方法参数就是不想写单独的实体类。这里宣告没有办法了。

又怎么办呢？我尝试了两种方法，第一种继承WebMvcConfigurer，重写addArgumentResolvers方法，可惜，传入的参数是一个空集合，要求你把你的自定义解析器放到集合里，最后再加到adapter的解析器集合里，仍然排在最后，没有出头之日。

最终办法只好重写了一下RequestMappingHandlerAdapter的afterPropertiesSet方法，手动在这里把我自定义的解析器放在解析器集合的第一个。然后配合我的自定义注解，避免影响到别人。最终实现了map参数里，value不全是string，也有integer的效果。

```java
    @Override
    public void afterPropertiesSet() {
        super.afterPropertiesSet();
        List<HandlerMethodArgumentResolver> argumentResolvers = getArgumentResolvers();
        if (argumentResolvers != null) {
            List<HandlerMethodArgumentResolver> argumentResolversort = new ArrayList<>();
            argumentResolversort.add(new MyRequestParamMapMethodArgumentResolver());
            argumentResolversort.addAll(argumentResolvers);
            setArgumentResolvers(argumentResolversort);
        }
    }
```