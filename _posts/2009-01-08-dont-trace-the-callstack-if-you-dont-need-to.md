---
title: Don't trace the callstack if you don't need to
categories : .Net
tags : Performance, Tracing
date: 2009-01-08 15:04:00 +10:00
---

Here is another performance tip with tracing. In configuration, there is the opportunity to define some tracing options. These options determine the actions taken by TraceListener when it writes the footer of the trace record for a given message. One of the options is to output the Callstack.   

{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8" ?> 
<configuration> 
  <system.diagnostics> 
    <trace useGlobalLock="false" /> 
    <sources> 
      <source name="WithCallstack" 
              switchValue="All"> 
        <listeners> 
          <clear /> 
          <add type="System.Diagnostics.DefaultTraceListener, System, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" 
              name="SlimListener" 
              traceOutputOptions="Callstack" /> 
        </listeners> 
      </source> 
      <source name="Slim" 
              switchValue="All"> 
        <listeners> 
          <clear /> 
          <add type="System.Diagnostics.DefaultTraceListener, System, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" 
              name="SlimListener" /> 
        </listeners> 
      </source> 
    </sources> 
  </system.diagnostics> 
</configuration> 

{% endhighlight %}

{% highlight csharp linenos %}
using System; 
using System.Diagnostics; 
using Microsoft.VisualStudio.TestTools.UnitTesting; 

namespace TestProject3 
{ 
    /// <summary> 
    /// Summary description for TraceTests 
    /// </summary> 
    [TestClass] 
    public class TraceTests 
    { 
        private const String Message = "this is the message to be written"; 
        private static readonly TraceSource CallstackSource = new TraceSource("WithCallstack"); 
        private static readonly TraceSource SlimSource = new TraceSource("Slim"); 

        /// <summary> 
        /// Callstacks the test. 
        /// </summary> 
        [TestMethod] 
        public void CallstackTest() 
        { 
            CallstackSource.TraceInformation(Message); 
        } 

        /// <summary> 
        /// Slims the test. 
        /// </summary> 
        [TestMethod] 
        public void SlimTest() 
        { 
            SlimSource.TraceInformation(Message); 
        } 

        /// <summary> 
        ///Gets or sets the test context which provides 
        ///information about and functionality for the current test run. 
        ///</summary> 
        public TestContext TestContext 
        { 
            get; 
            set; 
        } 
    } 
} 

{% endhighlight %}







