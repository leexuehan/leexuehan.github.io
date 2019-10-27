---
title: XWork 容器原理（三）
tags:
  - Struts2
date: 2015-07-27 21:38:32
categories: 技术
---

## Action

Action 是控制流元素的核心。Action的定义：

		public interface Action {
		
		    /**
		     * The action execution was successful. Show result
		     * view to the end user.
		     */
		    public static final String SUCCESS = "success";
		
		    /**
		     * The action execution was successful but do not
		     * show a view. This is useful for actions that are
		     * handling the view in another fashion like redirect.
		     */
		    public static final String NONE = "none";
		
		    /**
		     * The action execution was a failure.
		     * Show an error view, possibly asking the
		     * user to retry entering data.
		     */
		    public static final String ERROR = "error";
		
		    /**
		     * The action execution require more input
		     * in order to succeed.
		     * This result is typically used if a form
		     * handling action has been executed so as
		     * to provide defaults for a form. The
		     * form associated with the handler should be
		     * shown to the end user.
		     * <p/>
		     * This result is also used if the given input
		     * params are invalid, meaning the user
		     * should try providing input again.
		     */
		    public static final String INPUT = "input";
		
		    /**
		     * The action could not execute, since the
		     * user most was not logged in. The login view
		     * should be shown.
		     */
		    public static final String LOGIN = "login";
		
		
		    /**
		     * Where the logic of the action is executed.
		     *
		     * @return a string representing the logical result of the execution.
		     *         See constants in this interface for a list of standard result values.
		     * @throws Exception thrown if a system level exception occurs.
		     *                   <b>Note:</b> Application level exceptions should be handled by returning
		     *                   an error value, such as <code>Action.ERROR</code>.
		     */
		    public String execute() throws Exception;
		
		}

Action 接口的具体实现类是 XWork 进行核心业务逻辑处理的场所。

从源代码中可以看出，接口方法中并没有任何参数，表明Action执行时所需要的数据，无论是请求参数还是响应返回数据，都以Action属性变量的形式出现。

接口方法中返回值是一个字符串形式，表示业务逻辑是否执行成功的标志，在Action中起到流程控制的标志作用。

### Action 的突破点

1.**突破了传统的Servlet的模式中相应对象对Web容器的依赖**

任何响应对象只要实现了简单的Action接口，它就能成为一个Http响应类，我们在Http响应类中看不到任何对于Web容器的引用。

2.**突破了传统Servlet模式中响应对象无状态的限制**

XWork 中的Action是一个状态与动作的结合体。


## Interceptor

官方定义：

> Interceptor can execute code before and after an Action is invoked.

Interceptor 是属于 AOP 中的概念，本质是一个代码段，通过定义织入点来指定 Interceptor 的代码逻辑在织入点元素的之前或者之后执行，从而起到“拦截”的作用。

### 源码

		public interface Interceptor extends Serializable {
		
		    /**
		     * Called to let an interceptor clean up any resources it has allocated.
		     */
		    void destroy();
		
		    /**
		     * Called after an interceptor is created, but before any requests are processed using
		     * {@link #intercept(com.opensymphony.xwork2.ActionInvocation) intercept} , giving
		     * the Interceptor a chance to initialize any needed resources.
		     */
		    void init();
		
		    /**
		     * Allows the Interceptor to do some processing on the request before and/or after the rest of the processing of the
		     * request by the {@link ActionInvocation} or to short-circuit the processing and just return a String return code.
		     *
		     * @param invocation the action invocation
		     * @return the return code, either returned from {@link ActionInvocation#invoke()}, or from the interceptor itself.
		     * @throws Exception any system-level error, as defined in {@link com.opensymphony.xwork2.Action#execute()}.
		     */
		    String intercept(ActionInvocation invocation) throws Exception;
		
		}

init 和 destroy 方法在 Interceptor 对象初始化和销毁的时候被调用一次，主要用于对 Interceptor 内部所依赖的外部资源进行初始化和销毁管理。

Intercept 方法中的参数是 ActionInvocation。它是XWork控制流元素的核心调度元素。

在Intercept方法中使用这样一个元素作为参数，主要基于两点考虑：

1.便于 Interceptor 随时与控制流和数据流的其他元素沟通

ActionInvocation 的操作接口中不仅包含对控制流元素Action 和 ActionProxy 的访问控制，同时也包含了对数据流元素 ActionContext 和 ValueStack 的访问接口。

2.便于 ActionInvocation 在 Interceptor 的内部进行执行调度

在 Interceptor 内部，我们可以通过操作 ActionInvocation的调度接口，对整个控制流元素进行执行调度。

在Interceptor 内部，存在着两种不同的执行逻辑：

1.调用 ActionInvocation 的 invoke 方法来指定对执行栈的进一步调度执行。

Interceptor 在完成了自身的逻辑之后，有责任把执行的控制权转移到执行栈中的下一个元素继续执行，而执行栈的下一个元素不是 Interceptor 就是 Action 对象，因此他们又一次形成了一个自动**递归调用链**。

2.直接返回一个String类型的ResultCode 来终止执行栈的调度进行

## ActionInvocation

			public interface ActionInvocation extends Serializable {
			
			    /**
			     * Get the Action associated with this ActionInvocation.
			     *
			     * @return the Action
			     */
			    Object getAction();
			
			    /**
			     * Gets whether this ActionInvocation has executed before.
			     * This will be set after the Action and the Result have executed.
			     *
			     * @return <tt>true</tt> if this ActionInvocation has executed before.
			     */
			    boolean isExecuted();
			
			    /**
			     * Gets the ActionContext associated with this ActionInvocation. The ActionProxy is
			     * responsible for setting this ActionContext onto the ThreadLocal before invoking
			     * the ActionInvocation and resetting the old ActionContext afterwards.
			     *
			     * @return the ActionContext.
			     */
			    ActionContext getInvocationContext();
			
			    /**
			     * Get the ActionProxy holding this ActionInvocation.
			     *
			     * @return the ActionProxy.
			     */
			    ActionProxy getProxy();
			
			    /**
			     * If the ActionInvocation has been executed before and the Result is an instance of {@link ActionChainResult}, this method
			     * will walk down the chain of <code>ActionChainResult</code>s until it finds a non-chain result, which will be returned. If the
			     * ActionInvocation's result has not been executed before, the Result instance will be created and populated with
			     * the result params.
			     *
			     * @return the result.
			     * @throws Exception can be thrown.
			     */
			    Result getResult() throws Exception;
			
			    /**
			     * Gets the result code returned from this ActionInvocation.
			     *
			     * @return the result code
			     */
			    String getResultCode();
			
			 
			    void setResultCode(String resultCode);
			
			    /**
			     * Gets the ValueStack associated with this ActionInvocation.
			     *
			     * @return the ValueStack
			     */
			    ValueStack getStack();
			
			    /**
			     * Register a {@link PreResultListener} to be notified after the Action is executed and
			     * before the Result is executed.
			     * <p/>
			     * The ActionInvocation implementation must guarantee that listeners will be called in
			     * the order in which they are registered.
			     * <p/>
			     * Listener registration and execution does not need to be thread-safe.
			     *
			     * @param listener the listener to add.
			     */
			    void addPreResultListener(PreResultListener listener);
			
			    /**
			     * Invokes the next step in processing this ActionInvocation.
			     * <p/>
			     * If there are more Interceptors, this will call the next one. If Interceptors choose not to short-circuit
			     * ActionInvocation processing and return their own return code, they will call invoke() to allow the next Interceptor
			     * to execute. If there are no more Interceptors to be applied, the Action is executed.
			     * If the {@link ActionProxy#getExecuteResult()} method returns <tt>true</tt>, the Result is also executed.
			     *
			     * @throws Exception can be thrown.
			     * @return the return code.
			     */
			    String invoke() throws Exception;
			
			    /**
			     * Invokes only the Action (not Interceptors or Results).
			     * <p/>
			     * This is useful in rare situations where advanced usage with the interceptor/action/result workflow is
			     * being manipulated for certain functionality.
			     *
			     * @return the return code.
			     * @throws Exception can be thrown.
			     */
			    String invokeActionOnly() throws Exception;
			
			    /**
			     * Sets the action event listener to respond to key action events.
			     *
			     * @param listener the listener.
			     */
			    void setActionEventListener(ActionEventListener listener);
			
			    void init(ActionProxy proxy) ;
			
			    /**
			     * Prepares instance of ActionInvocation to be serializable,
			     * which simple means removing all unserializable fields, eg. Container
			     *
			     * @return ActionInvocation which can be serialize (eg. into HttpSession)
			     */
			    ActionInvocation serialize();
			
			    /**
			     * Performs opposite process to restore back ActionInvocation after deserialisation
			     *
			     * @param actionContext current {@link ActionContext}
			     * @return fully operational ActionInvocation
			     */
			    ActionInvocation deserialize(ActionContext actionContext);
			
			}

从上面的代码中可以看出，上面的接口方法可以分为以下几类：

1.对控制流元素和数据流元素的访问接口：getAction，getActionProxy，getStack

2.对执行调度流程的扩展接口：addPreListener，setActionEventListener

3.对执行栈进行调度执行的接口：invoke，invokeActionOnly


		/**
	     * @throws ConfigurationException If no result can be found with the returned code
	     */
	    public String invoke() throws Exception {
	        String profileKey = "invoke: ";
	        try {
	            UtilTimerStack.push(profileKey);
	
	            if (executed) {
	                throw new IllegalStateException("Action has already executed");
	            }
	
	            if (interceptors.hasNext()) {
	                final InterceptorMapping interceptor = interceptors.next();
	                String interceptorMsg = "interceptor: " + interceptor.getName();
	                UtilTimerStack.push(interceptorMsg);
	                try {
	                                resultCode = interceptor.getInterceptor().intercept(DefaultActionInvocation.this);
	                            }
	                finally {
	                    UtilTimerStack.pop(interceptorMsg);
	                }
	            } else {
	                resultCode = invokeActionOnly();
	            }
	
	            // this is needed because the result will be executed, then control will return to the Interceptor, which will
	            // return above and flow through again
	            if (!executed) {
	                if (preResultListeners != null) {
	                    for (Object preResultListener : preResultListeners) {
	                        PreResultListener listener = (PreResultListener) preResultListener;
	
	                        String _profileKey = "preResultListener: ";
	                        try {
	                            UtilTimerStack.push(_profileKey);
	                            listener.beforeResult(this, resultCode);
	                        }
	                        finally {
	                            UtilTimerStack.pop(_profileKey);
	                        }
	                    }
	                }
	
	                // now execute the result, if we're supposed to
	                if (proxy.getExecuteResult()) {
	                    executeResult();
	                }
	
	                executed = true;
	            }
	
	            return resultCode;
	        }
	        finally {
	            UtilTimerStack.pop(profileKey);
	        }
	    }

如果执行栈中下一个元素是 Interceptor 对象，那么执行该 Interceptor；如果执行栈中下一个元素是 Action 对象，那么执行该 Action；如果找不到下一个元素，执行终止，返回执行结果 ResultCode.

## ActionProxy

是位于整个XWork控制流体系的最外层的元素，也是整个XWork框架的门户。

		public interface ActionProxy {
		
		    /**
		     * Gets the Action instance for this Proxy.
		     *
		     * @return the Action instance
		     */
		    Object getAction();
		
		    /**
		     * Gets the alias name this ActionProxy is mapped to.
		     *
		     * @return the alias name
		     */
		    String getActionName();
		
		    /**
		     * Gets the ActionConfig this ActionProxy is built from.
		     *
		     * @return the ActionConfig
		     */
		    ActionConfig getConfig();
		
		    /**
		     * Sets whether this ActionProxy should also execute the Result after executing the Action.
		     *
		     * @param executeResult <tt>true</tt> to also execute the Result.
		     */
		    void setExecuteResult(boolean executeResult);
		
		    /**
		     * Gets the status of whether the ActionProxy is set to execute the Result after the Action is executed.
		     *
		     * @return the status
		     */
		    boolean getExecuteResult();
		
		    /**
		     * Gets the ActionInvocation associated with this ActionProxy.
		     *
		     * @return the ActionInvocation
		     */
		    ActionInvocation getInvocation();
		
		    /**
		     * Gets the namespace the ActionConfig for this ActionProxy is mapped to.
		     *
		     * @return the namespace
		     */
		    String getNamespace();
		
		    /**
		     * Execute this ActionProxy. This will set the ActionContext from the ActionInvocation into the ActionContext
		     * ThreadLocal before invoking the ActionInvocation, then set the old ActionContext back into the ThreadLocal.
		     *
		     * @return the result code returned from executing the ActionInvocation
		     * @throws Exception can be thrown.
		     * @see ActionInvocation
		     */
		    String execute() throws Exception;
		
		    /**
		     * Gets the method name to execute, or <tt>null</tt> if no method has been specified (meaning <code>execute</code> will be invoked).
		     *
		     * @return the method to execute
		     */
		    String getMethod();
		
		    /**
		     * Gets status of the method value's initialization.
		     *
		     * @return true if the method returned by getMethod() is not a default initializer value.
		     */
		    boolean isMethodSpecified();
		    
		}

可以从ActionProxy的定义中看到，ActionProxy与众多配置元素相关的操作接口。这表明：


> ActionProxy的首要职责是维护XWork的执行元素与请求对象之间的配置映射关系。

### ActionProxy的初始化

与众多的XWork内置元素一样，ActionProxy的构建过程也是通过工厂方法来完成的。

	public class DefaultActionProxyFactory implements ActionProxyFactory {
	
	    protected Container container;
	    
	    public DefaultActionProxyFactory() {
	        super();
	    }
	    
	    @Inject
	    public void setContainer(Container container) {
	        this.container = container;
	    }
	    
	    ...

	    public ActionProxy createActionProxy(String namespace, String actionName, String methodName, Map<String, Object> extraContext, boolean executeResult, boolean cleanupContext) {
	        //创建与ActionProxy所关联的ActionInvocation对象
	        ActionInvocation inv = new DefaultActionInvocation(extraContext, true);
			//依赖注入
	        container.inject(inv);
	        return createActionProxy(inv, namespace, actionName, methodName, executeResult, cleanupContext);
	    }
	   
		...

	    public ActionProxy createActionProxy(ActionInvocation inv, String namespace, String actionName, String methodName, boolean executeResult, boolean cleanupContext) {
	
	        DefaultActionProxy proxy = new DefaultActionProxy(inv, namespace, actionName, methodName, executeResult, cleanupContext);
	        container.inject(proxy);
	        proxy.prepare();
	        return proxy;
	    }
	
	    @Override
	    public String toString() {
	        return getClass().getSimpleName();
	    }
	
	}

## Action交互体系

### 外部环境交互体系

主要是指在XWork的Action中如何与XWork执行环境之外的调用环境进行交互。实际上就是Action与Web容器中的HttpServletRequest、HttpServletResponse、HttpSession对象的交互过程。

一种方法是调用ServletActionContext的静态方法：

	HttpServletRequest req = ServletActionContext.getRequest();
	
	HttpServletResponse res = ServletActionContext.getResponse();
	
	ServletContext sc  = ServletActionContext.getServletContext();

除此之外，Struts2还有另外一套 Web 对象的访问机制：

一套对象的访问机制首先建立在一系列的Aware接口之上。

	public interface ServletContextAware {
	
	    public void setServletContext(ServletContext context);
	}

若 Action 实现了上面所示的接口，我们就可以在这些由Aware接口所带来的设置外部环境操作接口方法中实现了这些操作接口的本地存储。

### 数据交互体系

反映了 Action 与数据流元素进行交互的过程。

Action 对于请求-响应的实现采用 POJO 模式，也就是说所有的请求数据和响应数据在 Action 中都以属性变量的形式出现，那么将请求数据从数据环境中读取，并设置到 Action 的属性变量的过程就是我们所说的映射过程。

这个过程是由 ParametersInterceptor 这个拦截器来完成。

		@Override
	    public String doIntercept(ActionInvocation invocation) throws Exception {
	        Object action = invocation.getAction();
	        if (!(action instanceof NoParameters)) {
	            ActionContext ac = invocation.getInvocationContext();
	            final Map<String, Object> parameters = retrieveParameters(ac);
	
	            if (LOG.isDebugEnabled()) {
	                LOG.debug("Setting params " + getParameterLogMap(parameters));
	            }
	
	            if (parameters != null) {
	                Map<String, Object> contextMap = ac.getContextMap();
	                try {
	                    ReflectionContextState.setCreatingNullObjects(contextMap, true);
	                    ReflectionContextState.setDenyMethodExecution(contextMap, true);
	                    ReflectionContextState.setReportingConversionErrors(contextMap, true);
	
	                    ValueStack stack = ac.getValueStack();
	                    setParameters(action, stack, parameters);
	                } finally {
	                    ReflectionContextState.setCreatingNullObjects(contextMap, false);
	                    ReflectionContextState.setDenyMethodExecution(contextMap, false);
	                    ReflectionContextState.setReportingConversionErrors(contextMap, false);
	                }
	            }
	        }
	        return invocation.invoke();
	    }

以上一段代码来自由于类 ParametersInterceptor.java。截取的是这个拦截器的骨架部分。提供了对于Action数据的全方位支持，包括 Action对象是否支持接受请求参数、请求参数的排序、过滤等等。这些功能支持都被封装到 setParameters 方法中。

	protected void setParameters(final Object action, ValueStack stack, final Map<String, Object> parameters) {
	        Map<String, Object> params;
	        Map<String, Object> acceptableParameters;
	        if (ordered) {
	            params = new TreeMap<String, Object>(getOrderedComparator());
	            acceptableParameters = new TreeMap<String, Object>(getOrderedComparator());
	            params.putAll(parameters);
	        } else {
	            params = new TreeMap<String, Object>(parameters);
	            acceptableParameters = new TreeMap<String, Object>();
	        }
	
	        for (Map.Entry<String, Object> entry : params.entrySet()) {
	            String name = entry.getKey();
	            Object value = entry.getValue();
	            if (isAcceptableParameter(name, action) && isAcceptableValue(value)) {
	                acceptableParameters.put(name, entry.getValue());
	            }
	        }
	
	        ValueStack newStack = valueStackFactory.createValueStack(stack);
	        boolean clearableStack = newStack instanceof ClearableValueStack;
	        if (clearableStack) {
	            //if the stack's context can be cleared, do that to prevent OGNL
	            //from having access to objects in the stack, see XW-641
	            ((ClearableValueStack)newStack).clearContextValues();
	            Map<String, Object> context = newStack.getContext();
	            ReflectionContextState.setCreatingNullObjects(context, true);
	            ReflectionContextState.setDenyMethodExecution(context, true);
	            ReflectionContextState.setReportingConversionErrors(context, true);
	
	            //keep locale from original context
	            context.put(ActionContext.LOCALE, stack.getContext().get(ActionContext.LOCALE));
	        }
	
	        boolean memberAccessStack = newStack instanceof MemberAccessValueStack;
	        if (memberAccessStack) {
	            //block or allow access to properties
	            //see WW-2761 for more details
	            MemberAccessValueStack accessValueStack = (MemberAccessValueStack) newStack;
	            accessValueStack.setAcceptProperties(acceptedPatterns.getAcceptedPatterns());
	            accessValueStack.setExcludeProperties(excludedPatterns.getExcludedPatterns());
	        }
	
	        for (Map.Entry<String, Object> entry : acceptableParameters.entrySet()) {
	            String name = entry.getKey();
	            Object value = entry.getValue();
	            try {
	                newStack.setParameter(name, value);
	            } catch (RuntimeException e) {
	                if (devMode) {
	                    notifyDeveloperParameterException(action, name, e.getMessage());
	                }
	            }
	        }
	
	        if (clearableStack && (stack.getContext() != null) && (newStack.getContext() != null))
	            stack.getContext().put(ActionContext.CONVERSION_ERRORS, newStack.getContext().get(ActionContext.CONVERSION_ERRORS));
	
	        addParametersToContext(ActionContext.getContext(), acceptableParameters);
	    }
	
	    protected void notifyDeveloperParameterException(Object action, String property, String message) {
	        String developerNotification = LocalizedTextUtil.findText(ParametersInterceptor.class, "devmode.notification",
	                ActionContext.getContext().getLocale(), "Developer Notification:\n{0}",
	                new Object[]{
	                        "Unexpected Exception caught setting '" + property + "' on '" + action.getClass() + ": " + message
	                }
	        );
	        LOG.error(developerNotification);
	        // see https://issues.apache.org/jira/browse/WW-4066
	        if (action instanceof ValidationAware) {
	            Collection<String> messages = ((ValidationAware) action).getActionMessages();
	            messages.add(message);
	            ((ValidationAware) action).setActionMessages(messages);
	        }
	    }

> ParametersInterceptor 是一个将外部环境数据与 Action 的属性变量进行融合的骨架，而其中的 retrieveParameters 方法，就是外部环境数据的提供者。具体来说，只要继承ParametersInterceptor，然后覆盖其中的retrieveParameters方法，将其替换为自己的扩展逻辑，就可以轻松实现 Action 对象的数据交互。


	protected Map<String, Object> retrieveParameters(ActionContext ac) {
        return ac.getParameters();
    }

### 协作交互体系

略。