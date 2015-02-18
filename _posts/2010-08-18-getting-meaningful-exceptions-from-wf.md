---
title: Getting meaningful exceptions from WF
categories : .Net
tags : WF
date: 2010-08-18 13:33:00 +10:00
---

Overall I love the changes made to WF4. The latest workflow support is a huge leap forward from the 3.x versions. Unfortunately both versions suffer the same issues regarding feedback for the developer when exceptions are thrown. 

Consider the following simple workflow example.

![image][0]

{% highlight csharp linenos %}
namespace WorkflowConsoleApplication1
{
    using System;
    using System.Activities;
    class Program
    {
        static void Main(string[] args)
        { 
            try
            {
                WorkflowInvoker.Invoke(new Workflow1());
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
            }
            Console.ReadKey();
        }
    }
}
{% endhighlight %}

This example provides the following output.

{% highlight text linenos %}
System.InvalidOperationException: Something went wrong
   at System.Activities.WorkflowApplication.Invoke(Activity activity, IDictionar
y`2 inputs, WorkflowInstanceExtensionManager extensions, TimeSpan timeout)
   at System.Activities.WorkflowInvoker.Invoke(Activity workflow, TimeSpan timeo
ut, WorkflowInstanceExtensionManager extensions)
   at System.Activities.WorkflowInvoker.Invoke(Activity workflow)
   at WorkflowConsoleApplication1.Program.Main(String[] args) in e:\users\profil
edr\documents\visual studio 2010\Projects\WorkflowConsoleApplication2\WorkflowCo
nsoleApplication2\Program.cs:line 12
{% endhighlight %}


Debugging WF activities with this constraint is very difficult. Trying to debug your workflows gets exponentially harder with increasing complexity of the workflow system and it's underlying components. This is compounded when the exception (such as a NullReferenceException) doesn't provide enough information by itself to pinpoint where the exception was thrown.





{% highlight csharp linenos %}
namespace WorkflowConsoleApplication1
{
    using System;
    using System.Activities;
    using System.Collections.Generic;
    using System.Reflection;
    using System.Text;
    using System.Threading;
    internal class ActivityInvoker
    {
        private static readonly MethodInfo _preserveStackTraceMethod = GetExceptionPreserveMethod();
        public static IDictionary<String, Object> Invoke(Activity activity, IDictionary<String, Object> inputParameters = null)
        {
            if (inputParameters == null)
            {
                inputParameters = new Dictionary<String, Object>();
            }
            WorkflowApplication application = new WorkflowApplication(activity, inputParameters);
            Exception thrownException = null;
            IDictionary<String, Object> outputParameters = new Dictionary<String, Object>();
            ManualResetEvent waitHandle = new ManualResetEvent(false);
            application.OnUnhandledException = (WorkflowApplicationUnhandledExceptionEventArgs arg) =>
            {
                // Preserve the stack trace in this exception
                // This is a hack into the Exception.InternalPreserveStackTrace method that allows us to re-throw and preserve the call stack
                _preserveStackTraceMethod.Invoke(arg.UnhandledException, null);
                thrownException = arg.UnhandledException;
                return UnhandledExceptionAction.Terminate;
            };
            application.Completed = (WorkflowApplicationCompletedEventArgs obj) =>
            {
                waitHandle.Set();
                outputParameters = obj.Outputs;
            };
            application.Aborted = (WorkflowApplicationAbortedEventArgs obj) => waitHandle.Set();
            application.Idle = (WorkflowApplicationIdleEventArgs obj) => waitHandle.Set();
            application.PersistableIdle = (WorkflowApplicationIdleEventArgs arg) =>
            {
                waitHandle.Set();
                return PersistableIdleAction.Persist;
            };
            application.Unloaded = (WorkflowApplicationEventArgs obj) => waitHandle.Set();
            application.Run();
            waitHandle.WaitOne();
            if (thrownException != null)
            {
                throw thrownException;
            }
            return outputParameters;
        }
        private static MethodInfo GetExceptionPreserveMethod()
        {
            return typeof(Exception).GetMethod("InternalPreserveStackTrace", BindingFlags.Instance | BindingFlags.NonPublic);
        }
    }
}
{% endhighlight %}


{% highlight text linenos %}
System.InvalidOperationException: Something went wrong
   at System.Activities.Statements.Throw.Execute(CodeActivityContext context)
   at System.Activities.CodeActivity.InternalExecute(ActivityInstance instance,
ActivityExecutor executor, BookmarkManager bookmarkManager)
   at System.Activities.ActivityInstance.Execute(ActivityExecutor executor, Book
markManager bookmarkManager)
   at System.Activities.Runtime.ActivityExecutor.ExecuteActivityWorkItem.Execute
Body(ActivityExecutor executor, BookmarkManager bookmarkManager, Location result
Location)
   at WorkflowConsoleApplication1.ActivityInvoker.Invoke(Activity activity, IDic
tionary`2 inputParameters) in e:\users\profiledr\documents\visual studio 2010\Pr
ojects\WorkflowConsoleApplication2\WorkflowConsoleApplication2\ActivityInvoker.c
s:line 82
   at WorkflowConsoleApplication1.Program.Main(String[] args) in e:\users\profil
edr\documents\visual studio 2010\Projects\WorkflowConsoleApplication2\WorkflowCo
nsoleApplication2\Program.cs:line 11
{% endhighlight %}


    
    
namespace WorkflowConsoleApplication1
{
    using System;
    using System.Activities;
    using System.Collections.Generic;
    using System.Reflection;
    using System.Text;
    using System.Threading;
    internal class ActivityInvoker
    {
        private static readonly PropertyInfo _parentProperty = GetActivityParentProperty();
        private static readonly MethodInfo _preserveStackTraceMethod = GetExceptionPreserveMethod();
        public static Exception CreateActivityFailureException(Exception exception, Activity source)
        {
            // Create the hierarchy of activity names
            Activity activity = source;
            StringBuilder builder = new StringBuilder(exception.Message);
            builder.AppendLine();
            builder.AppendLine();
            builder.AppendLine("Workflow exception throw from the following activity stack:");
            while (activity != null)
            {
                builder.AppendLine(activity + " - " + activity.GetType().FullName);
                activity = _parentProperty.GetValue(activity, null) as Activity;
            }
            return new ActivityFailureException(builder.ToString(), exception);
        }
        public static IDictionary<String, Object> Invoke(Activity activity, IDictionary<String, Object> inputParameters = null)
        {
            if (inputParameters == null)
            {
                inputParameters = new Dictionary<String, Object>();
            }
            WorkflowApplication application = new WorkflowApplication(activity, inputParameters);
            Exception thrownException = null;
            IDictionary<String, Object> outputParameters = new Dictionary<String, Object>();
            ManualResetEvent waitHandle = new ManualResetEvent(false);
            application.OnUnhandledException = (WorkflowApplicationUnhandledExceptionEventArgs arg) =>
            {
                // Preserve the stack trace in this exception
                // This is a hack into the Exception.InternalPreserveStackTrace method that allows us to re-throw and preserve the call stack
                _preserveStackTraceMethod.Invoke(arg.UnhandledException, null);
                thrownException = CreateActivityFailureException(arg.UnhandledException, arg.ExceptionSource);
                return UnhandledExceptionAction.Terminate;
            };
            application.Completed = (WorkflowApplicationCompletedEventArgs obj) =>
            {
                waitHandle.Set();
                outputParameters = obj.Outputs;
            };
            application.Aborted = (WorkflowApplicationAbortedEventArgs obj) => waitHandle.Set();
            application.Idle = (WorkflowApplicationIdleEventArgs obj) => waitHandle.Set();
            application.PersistableIdle = (WorkflowApplicationIdleEventArgs arg) =>
            {
                waitHandle.Set();
                return PersistableIdleAction.Persist;
            };
            application.Unloaded = (WorkflowApplicationEventArgs obj) => waitHandle.Set();
            application.Run();
            waitHandle.WaitOne();
            if (thrownException != null)
            {
                throw thrownException;
            }
            return outputParameters;
        }
        private static PropertyInfo GetActivityParentProperty()
        {
            return typeof(Activity).GetProperty("Parent", BindingFlags.Instance | BindingFlags.GetProperty | BindingFlags.NonPublic);
        }
        private static MethodInfo GetExceptionPreserveMethod()
        {
            return typeof(Exception).GetMethod("InternalPreserveStackTrace", BindingFlags.Instance | BindingFlags.NonPublic);
        }
    }
}
{% endhighlight %}



{% highlight text linenos %}
WorkflowConsoleApplication1.ActivityFailureException: Something went wrong
Workflow exception thrown from the following activity stack:
1.5: Throw - System.Activities.Statements.Throw
1.4: Fourth Sequence - System.Activities.Statements.Sequence
1.3: Third Sequence - System.Activities.Statements.Sequence
1.2: Second Sequence - System.Activities.Statements.Sequence
1.1: First Sequence - System.Activities.Statements.Sequence
1: Workflow1 - WorkflowConsoleApplication1.Workflow1
 ---> System.InvalidOperationException: Something went wrong
   at System.Activities.Statements.Throw.Execute(CodeActivityContext context)
   at System.Activities.CodeActivity.InternalExecute(ActivityInstance instance,
ActivityExecutor executor, BookmarkManager bookmarkManager)
   at System.Activities.ActivityInstance.Execute(ActivityExecutor executor, Book
markManager bookmarkManager)
   at System.Activities.Runtime.ActivityExecutor.ExecuteActivityWorkItem.Execute
Body(ActivityExecutor executor, BookmarkManager bookmarkManager, Location result
Location)
   --- End of inner exception stack trace ---
   at WorkflowConsoleApplication1.ActivityInvoker.Invoke(Activity activity, IDic
tionary`2 inputParameters) in e:\users\profiledr\documents\visual studio 2010\Pr
ojects\WorkflowConsoleApplication2\WorkflowConsoleApplication2\ActivityInvoker.c
s:line 80
   at WorkflowConsoleApplication1.Program.Main(String[] args) in e:\users\profil
edr\documents\visual studio 2010\Projects\WorkflowConsoleApplication2\WorkflowCo
nsoleApplication2\Program.cs:line 11
{% endhighlight %}
    

[0]: /blogfiles/image_25.png