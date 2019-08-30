---
layout: post
title: Batteries not included.
image: https://www.parisi.io/wp-content/uploads/2018/01/Screen-Shot-2018-01-19-at-1.21.09-PM-1024x305.png
---

<h1>Managing Resources</h1>
I've been hacking away at a robotic car built with a RaspberryPi. I had no way of monitoring the battery life, so it occurred to me that we need such a mechanism in MiNiFi C++. Configuring MiNiFi C++ for the correct number of threads and timing can be difficult, therefore we've created a controller service that can monitor battery life in Linux in order to adjust the thread pool settings. The controller service is titled LinuxPowerManagerService. It can be configured as we see in the image below. LinuxPowerManagerService is the first controller service that functions as a proof of concept to monitor and augment thread pools within the agent in response to battery capacity and status. To configure this, we specify the battery capacity and status paths, along with the trigger and low battery thresholds. The trigger threshold is the threshold of the battery capacity before we begin reducing threads and incurring wait in our scheduling agents between processor executions. The low battery threshold is a threshold at which we respond more aggressively to reducing resources consumed.
<figure class="caption">
<p >
<img class="aligncenter size-large wp-image-370" src="https://www.parisi.io/wp-content/uploads/2018/01/Screen-Shot-2018-01-19-at-1.21.09-PM-1024x305.png" alt="" width="1024" height="305" />
</p>
</figure>
The battery status is a path that specifies the current state of the battery. State is typically defined as charging or discharging. This is important as the LinuxPowerManagerService makes an attempt to reduce resources if and only if we are still in a discharged state. The wait period is the frequency at which we will make adjustments the internal MiNiFi C++ threadpools. If the low battery threshold is not met then this wait period will also be the period in which we will make any adjustments to thread resources.
<h1>Avoiding Starvation</h1>
The TheadPoolManager controller service requires that we do not starve processors. In testing I found that the reduction in thread pool threads and increased time slicing that we incur with the manager results in increased yield in the flow. This may or may not be desirable in our flow. Testing concluded that as we reduce our thread pool we see a reduction in speed. The controller service will avoid starvation by leaving one thread available to do all work. This means that if you have a simple flow ( say GenerateFlowFile --> LogAttribute ) your reduction in CPU consumed may be up to 30% of a rather small amount.
<h1>Recovering</h1>
Below I've attempted to demonstrate the reduction in threads followed by the increase in the number of threads in htop. The increase originates from plugging the battery in, resulting in a charging state. The second image shows us after ten minutes of execution once we've lowered below our threshold. The final image demonstrates the increase in threads after I accidentally plugged the computer back in. Once charging, the agent will increase the number of threads incrementally.  If the agent enters into a discharge state, it will immediately resume reduction of threads from the current state.

<figure class="caption">
<p ><img class="wp-image-379 size-large" src="https://www.parisi.io/wp-content/uploads/2018/01/Screen-Shot-2018-01-19-at-1.39.14-PM-1024x290.png" alt="" width="1024" height="290" /><figcaption>Demonstrating Threads</figcaption></p></figure>

<figure class="caption">
<p ><img class="wp-image-377 size-large" src="https://www.parisi.io/wp-content/uploads/2018/01/Screen-Shot-2018-01-19-at-1.47.02-PM-1024x292.png" alt="" width="1024" height="292" /><figcaption>Reducing Threads</figcaption></p></figure>

<figure class="caption">
<p ><img class="wp-image-378 size-large" src="https://www.parisi.io/wp-content/uploads/2018/01/Screen-Shot-2018-01-19-at-1.50.00-PM-1024x301.png" alt="" width="1024" height="301" /><figcaption>Increasing threads while charging</figcaption></p></figure>
<h1>Battery Management in action</h1>
<figure class="caption">
<p >
<iframe width="560" height="315" src="https://www.youtube.com/embed/i2j6XVgonkM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
<figcaption>Demonstrating Threads</figcaption></p></figure>
<h1>Conclusion</h1>
In this article we've take a quick dive into monitoring battery state and capacity. Capacity is intended to be the current energy level of our battery(ies). If we reach our threshold the agent will automatically reduce the number of threads devoted to processor execution. In doing so we'll also see an increase in the sleep and yield times between processor onTrigger calls.
