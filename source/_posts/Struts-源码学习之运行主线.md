---
title: Struts 源码学习之运行主线
tags:
  - Struts2
date: 2015-07-30 22:59:10
categories: 技术
---

首先还是要把上一篇那个过滤器架构图挪用过来

![](http://i.imgur.com/rFiry1D.png)

在初始化主线时介绍了 init 方法，则当用之来处理 Http 请求时，则意味着进入运行主线，于是就该介绍 doFilter方法了。

![](http://i.imgur.com/3GxyRvc.png)

在进行 URL 处理时，doFilter 方法会根据职责的不同，有条不紊地将URL处理过程分配给 PrepareOperations 和 ExecuteOperations 来分别完成。但是经过前面的分析，我们知道 它们内部封装了 Dispatcher 的对象，所以实质上还是由 Dispatcher 来完成的。下面是 PrepareOperations 中的一个方法，可以看到其内部的执行还是通过 Dispatcher 对象来完成，很好地印证了这一点。

	public void setEncodingAndLocale(HttpServletRequest request, HttpServletResponse response) {
        dispatcher.prepare(request, response);
    }

### createActionContext() 方法

![](http://i.imgur.com/AFUHUa9.png)

### assignDispatcherToThread() 方法

	public void assignDispatcherToThread() {
        Dispatcher.setInstance(dispatcher);
    }

将 dispatcher 对象绑定到当前线程。

### 封装 HttpServletRequest 

	public HttpServletRequest wrapRequest(HttpServletRequest oldRequest) throws ServletException {
        HttpServletRequest request = oldRequest;
        try {
            // Wrap request first, just in case it is multipart/form-data
            // parameters might not be accessible through before encoding (ww-1278)
            request = dispatcher.wrapRequest(request);
        } catch (IOException e) {
            throw new ServletException("Could not wrap servlet request with MultipartRequestWrapper!", e);
        }
        return request;
    }

这里对其进行封装的还是 dispatcher。下面是 dispatcher 的封装方法：

	public HttpServletRequest wrapRequest(HttpServletRequest request) throws IOException {
        // don't wrap more than once
        if (request instanceof StrutsRequestWrapper) {
            return request;
        }

        String content_type = request.getContentType();
        if (content_type != null && content_type.contains("multipart/form-data")) {
            MultiPartRequest mpr = getMultiPartRequest();
            LocaleProvider provider = getContainer().getInstance(LocaleProvider.class);
            request = new MultiPartRequestWrapper(mpr, request, getSaveDir(), provider, disableRequestAttributeValueStackLookup);
        } else {
            request = new StrutsRequestWrapper(request, disableRequestAttributeValueStackLookup);
        }

        return request;
    }
	

注意到其中有个 StrutsRequestWrapper 方法，这个方法采用了装饰器模式实现。

	public class StrutsRequestWrapper extends HttpServletRequestWrapper {
	
	    private static final String REQUEST_WRAPPER_GET_ATTRIBUTE = "__requestWrapper.getAttribute";
	    private final boolean disableRequestAttributeValueStackLookup;
	
	    /**
	     * The constructor
	     * @param req The request
	     */
	    public StrutsRequestWrapper(HttpServletRequest req) {
	        this(req, false);
	    }
	
	    /**
	     * The constructor
	     * @param req The request
	     * @param disableRequestAttributeValueStackLookup flag for disabling request attribute value stack lookup (JSTL accessibility)
	     */
	    public StrutsRequestWrapper(HttpServletRequest req, boolean disableRequestAttributeValueStackLookup) {
	        super(req);
	        this.disableRequestAttributeValueStackLookup = disableRequestAttributeValueStackLookup;
	    }
	
	    /**
	     * Gets the object, looking in the value stack if not found
	     *
	     * @param key The attribute key
	     */
	    public Object getAttribute(String key) {
	        if (key == null) {
	            throw new NullPointerException("You must specify a key value");
	        }
	
	        if (disableRequestAttributeValueStackLookup || key.startsWith("javax.servlet")) {
	            // don't bother with the standard javax.servlet attributes, we can short-circuit this
	            // see WW-953 and the forums post linked in that issue for more info
	            return super.getAttribute(key);
	        }
	
	        ActionContext ctx = ActionContext.getContext();
	        Object attribute = super.getAttribute(key);
	
	        if (ctx != null && attribute == null) {
	            boolean alreadyIn = isTrue((Boolean) ctx.get(REQUEST_WRAPPER_GET_ATTRIBUTE));
	
	            // note: we don't let # come through or else a request for
	            // #attr.foo or #request.foo could cause an endless loop
	            if (!alreadyIn && !key.contains("#")) {
	                try {
	                    // If not found, then try the ValueStack
	                    ctx.put(REQUEST_WRAPPER_GET_ATTRIBUTE, Boolean.TRUE);
	                    ValueStack stack = ctx.getValueStack();
	                    if (stack != null) {
	                        attribute = stack.findValue(key);
	                    }
	                } finally {
	                    ctx.put(REQUEST_WRAPPER_GET_ATTRIBUTE, Boolean.FALSE);
	                }
	            }
	        }
	        return attribute;
	    }
	}

为什么要这么大费周章地对其进行封装呢？主要原因在于：


> 传统的Web容器元素的数据都是通过 HttpServletRequest 的接口实现访问的；而在 Struts2 中，数据源的存储位置发生了变化，它们变得不再与 Web 容器绑定在一起，而是以 ActionContext 的形式存在于当前线程中。因此，传统的访问方式将无法访问 Struts2 中的数据。

根据 Http 请求查找 ActionMapping

	public ActionMapping findActionMapping(HttpServletRequest request, HttpServletResponse response, boolean forceLookup) {
        ActionMapping mapping = (ActionMapping) request.getAttribute(STRUTS_ACTION_MAPPING_KEY);
        if (mapping == null || forceLookup) {
            try {
                mapping = dispatcher.getContainer().getInstance(ActionMapper.class).getMapping(request, dispatcher.getConfigurationManager());
                if (mapping != null) {
                    request.setAttribute(STRUTS_ACTION_MAPPING_KEY, mapping);
                }
            } catch (Exception ex) {
                dispatcher.sendError(request, response, HttpServletResponse.SC_INTERNAL_SERVER_ERROR, ex);
            }
        }

        return mapping;
    }

注意到有一个ActionMapping 的对象。翻开其源码可以看出其内部构造：

![](http://i.imgur.com/qeVvhYM.png)

可以看出来，其内部本身并不包含逻辑，它只是一个简单的 JavaBean 而已。其实质上表示 URL 与 Java 对象对应关系的一个组件类。

在Struts2进行Http请求处理时，由 ActionMapper 的实现类在运行期间查找相应事件映射关系并生成 ActionMapping 对象。

**ActionMapper 才是 Struts2 进行 URL Mapping 关系查找的核心类。**

![](http://i.imgur.com/HRvo1oI.png)

上面是一个 ActionMapper 的默认实现类，可以从中看到此类提供了很多查找策略。因而，ActionMapper 成为Struts2 的一个重要的扩展点，我们可以自己实现 ActionMapper 从而指定各种不同的 URL Mapping的实现方式。

## ExecuteOperations

从上一篇可以看到 ExecuteOperations 的结构。在 ExecuteOperations 中仅仅包含两个方法，分别用于处理静态资源和实际的 Http 请求。

其中核心方法是 serviceAction 方法，这个方法是属于 Dispatcher 的。

	public void serviceAction(HttpServletRequest request, HttpServletResponse response, ActionMapping mapping)
            throws ServletException {
		//创建 MVC 的运行数据环境
        Map<String, Object> extraContext = createContextMap(request, response, mapping);

        // If there was a previous value stack, then create a new copy and pass it in to be used by the new Action
        ValueStack stack = (ValueStack) request.getAttribute(ServletActionContext.STRUTS_VALUESTACK_KEY);
        boolean nullStack = stack == null;
		//没有找到已经存在的 ValueStack，则从 ActionContext 中获取当前线程的 ValueStack
        if (nullStack) {
            ActionContext ctx = ActionContext.getContext();
            if (ctx != null) {
				//ActionContext是线程安全的，所以这里获取的是当前线程的 ValueStack
                stack = ctx.getValueStack();
            }
        }
		//将 ValueStack 也设置到当前的数据环境中
        if (stack != null) {
            extraContext.put(ActionContext.VALUE_STACK, valueStackFactory.createValueStack(stack));
        }

        String timerKey = "Handling request from Dispatcher";
        try {
            UtilTimerStack.push(timerKey);
			//从ActionMapping 中分别获取 Action 的 3 大元素
            String namespace = mapping.getNamespace();
            String name = mapping.getName();
            String method = mapping.getMethod();
			//创建一个 ActionProxy，从此已经进入 XWork的世界了
            ActionProxy proxy = getContainer().getInstance(ActionProxyFactory.class).createActionProxy(
                    namespace, name, method, extraContext, true, false);

            request.setAttribute(ServletActionContext.STRUTS_VALUESTACK_KEY, proxy.getInvocation().getStack());

            // if the ActionMapping says to go straight to a result, do it!
            if (mapping.getResult() != null) {
                Result result = mapping.getResult();
                result.execute(proxy.getInvocation());
            } else {
				// 执行 ActionProxy，真正运行 XWork 中的MVC实现
                proxy.execute();
            }

            // If there was a previous value stack then set it back onto the request
            if (!nullStack) {
                request.setAttribute(ServletActionContext.STRUTS_VALUESTACK_KEY, stack);
            }
        } catch (ConfigurationException e) {
            logConfigurationException(request, e);
            sendError(request, response, HttpServletResponse.SC_NOT_FOUND, e);
        } catch (Exception e) {
            if (handleException || devMode) {
                sendError(request, response, HttpServletResponse.SC_INTERNAL_SERVER_ERROR, e);
            } else {
                throw new ServletException(e);
            }
        } finally {
            UtilTimerStack.pop(timerKey);
        }
    }

从上面可以看到，XWork 框架的入口类： **ActionProxy**。

可以得出有关 Dispatcher 这个核心分发器的重要结论：

> 1.XWork 框架逻辑实际上由 Dispatcher 创建并负责驱动执行。
> 
> 2.Dispatcher 负责Http请求不同处理阶段的数据转换工作
> 
> 3.在 ActionProxy 中，我们将不再看到任何与Web容器相关的对象。

因此，Dispatcher是真正将 Http请求与 MVC 实现分离的核心分发器。

ActionProxy的创建过程：

	ActionProxy proxy = getContainer().getInstance(ActionProxyFactory.class).createActionProxy(
                    namespace, name, method, extraContext, true, false);

从上面的创建方法中可以看到参数有：

**配置关系映射**——namespace、actionname、method等；

这三个参数都是来源于 ActionMapping 对象。

**运行上下文环境**——extraContext。

这两类参数构成了XWork执行环境的数据基础。


ActionInvocation 的初始化过程由 ActionProxy 驱动完成。

	public void init(ActionProxy proxy) {
        this.proxy = proxy;
		//创建上下文环境，这里的contextMap 与 ActionContext 的上下文环境一致。
        Map<String, Object> contextMap = createContextMap();

        // Setting this so that other classes, like object factories, can use the ActionProxy and other
        // contextual information to operate
        ActionContext actionContext = ActionContext.getContext();

        if (actionContext != null) {
            actionContext.setActionInvocation(this);
        }
		//创建 Action 对象
        createAction(contextMap);
		//将Action对象置于ValueStack
		//这就是将 XWork 的数据流元素与控制流元素进行融合的关键步骤
        if (pushAction) {
            stack.push(action);
            contextMap.put("action", action);
        }

		//构建ActionInvocation的上下文环境
		//将ActionInvocation的上下文环境与 ActionContext 等同起来。
        invocationContext = new ActionContext(contextMap);
        invocationContext.setName(proxy.getActionName());

        // 获取拦截器堆栈，并将拦截器堆栈的迭代器指针处于栈顶位置
        List<InterceptorMapping> interceptorList = new ArrayList<InterceptorMapping>(proxy.getConfig().getInterceptors());
        interceptors = interceptorList.iterator();
    }
