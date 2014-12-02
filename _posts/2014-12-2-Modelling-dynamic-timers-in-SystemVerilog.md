---
layout: post
author: geeta_yadav
title: Modelling Dynamic Timers in SystemVerilog
excerpt: ""
modified: 2014-12-02
tags: [systemverilog, disable fork, timer, keepalive]
comments: true
image:
  feature: #sample-image-5.jpg
  credit: #WeGraphics
  creditlink: #http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/
---



In embedded systems when testbenching certain scenarios, we often come across situations where we need to keep a tab on a packet or signal appearing in a periodic fashion. One such scenario is the [http://en.wikipedia.org/wiki/Keepalive](keepalive) ethernet packet that signals an ethernet device to keep a routing path in consideration. While at signal level you might make use of temporal assertions, a keepalive packet scenario offers an altogether different challenge. For one, temporal assertions are naive and cannot detect packet structures and formats. Additionally, assertions and signals they monitor are static in nature. On the other hand a multitude of links can be formed and broken during a given simulation. The contents of a keepalive packet are generally used for indicating which link they are associated with. To detect and monitor a keepalive packet in the testbench, we obviously have to work at transaction level. Though keepalive protocol is used at various layers of a network stack, in this blog i will focus on a basic ethernet keepalive packet that identifies with the packets origin using the source MAC address. Such a monitor can be used to track and monitor various devices plugged into a router. A popular verification site [http://www.testbench.in/TB_21_WATCHDOG.html] has given a good `module` based timer example but i want to make a dynamic (class based) that can be instantiated for multiple keepalive scenarios. For that we will use systemverilog class which allows dynamic creation and deletion of objects. So, now you have a Timer class, a reset function inside the class which has two threads running in parallel, when event is triggered the timer is reset and in the other thread if a ethernet packet is not received for certain duration of time Watchdog is activated.
	
{% highlight systemverilog linenos %}
class Timer;
   event trigger;
   longint source_addr;
   time    duration;
  
  function new(time d);
    duration=d;
  endfunction // new
  
  function void reset();
    fork
      begin
	->trigger;
	fork:rst	       
	  begin
	    wait (trigger);
	    $display("Timer Reset");
	  end
	  begin
	    #duration;
	    $display("Watchdog activated after", duration);
	  end
	join_any
	disable rst;
      end // fork begin
    join_none
  endfunction // reset
endclass // Timer
{% endhighlight %}

Now, lets create a single instance of class `Timer`: 

{% highlight systemverilog linenos %}
module top;
   Timer t1;
   initial
     begin
       t1=new(5);
       fork	
	     t1.reset();
       join_none
       $display("Top module fork joined");
       #30 $finish;
     end
endmodule // top
{% endhighlight %}

We get this:

```
# Top module fork joined
# Watchdog activated after	5
# $finish

```
Ok, it works perfectly as i wanted. Lets check disable rst functionality for multiple instances `t1` and  `t2`:

{% highlight systemverilog linenos %}
module top;
   Timer t1;
   Timer t2;
   initial
     begin
       t1=new(5);
       t2=new(10);
       fork	
	     t1.reset();
	     t2.reset();
       join_none
       $display("Top module fork joined");
       #30 $finish;
     end
endmodule // top
{% endhighlight %}

Output:

```
# Top module fork joined
# Watchdog activated after	5
# $finish  

```

This is not what we wanted. `Timer` terminates fork for `t2` as well because labelled disable forks in SystemVerilog are statically scoped. SystemVerilog provides another way to disable a fork using the construct `disable fork`. This construct disables all the child threads of a running thread:

{% highlight systemverilog linenos %}
class Timer;
   event trigger;
   longint source_addr;
   time    duration;
  
  function new(time d);
    duration=d;
  endfunction // new
  
  function void reset();
    fork
      begin
	->trigger;
	fork:rst	       
	  begin
	    wait (trigger);
	    $display("Timer Reset");
	  end
	  begin
	    #duration;
	    $display("Watchdog activated after", duration);
	  end
	join_any
	disable fork;
      end // fork begin
    join_none
  endfunction // reset
endclass // Timer
{% endhighlight %}

Now when we instantiate the `Timer` multiple times, we get the desired results:

{% highlight systemverilog linenos %}
module top;
   Timer t1;
   Timer t2;
   initial
     begin
       t1=new(5);
       t2=new(10);
       fork	
	     t1.reset();
	     t2.reset();
       join_none
       $display("Top module fork joined");
       #30 $finish;
     end
endmodule // top
{% endhighlight %}

Check out the output:

```

# Top module fork joined
# Watchdog activated after	5
# Watchdog activated after	10
# $finish

```
