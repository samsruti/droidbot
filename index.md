---
layout: default
title: DroidBot by lynnlyc
---

# DroidBot

## Table of Content

+ [Introduction](#introduction)
+ [How does DroidBot work](#how-does-droidbot-work)
+ [Installation](#installation)
+ [Usage](#usage)
+ [Testing](#testing)
+ [Limitations](#limitations)
+ [List of Posts](#list-of-posts)

## Introduction
DroidBot is an Android app exerciser.

It automatically interacts with Android app by sending user events such as gestures, broadcasts and key presses.
It is useful in situations where automatic testing is needed. 
For example, testing Android app in an online sandbox, or testing a large amount of apps.

It does the similar thing as [monkey](http://developer.android.com/tools/help/monkey.html), but is much smarter:

1. DroidBot can get a list of sensitive user events from static analysis of Android App;
2. DroidBot can avoid redundant reentries of exploited UI states by dynamic UI analysis.
3. DroidBot does not send random gestures (touch, click etc.) like monkey, instead, 
it send specific gestures according to the position and type of UI element.

In a word, DroidBot does a lot of things to **improve testing coverage**.
In malware detection, higher coverage often leads to finding more sensitive behaviors.

We integrated DroidBot with [DroidBox](https://github.com/pjlantz/droidbox)
and evaluated DroidBot by comparing with monkey. 
The result demonstrate that DroidBot is better than monkey on detecting more sensitive behaviors.

## How does DroidBot work
DroidBot follows four steps when testing an app.

1. Connect to a device. The device could be an emulator (DroidBox in particular) or a real device. 
According to the device type, DroidBot establishes the proper connections (usually adb and telnet).
2. Analyse the apk sample. DroidBot runs static analysis of apk using androguard, 
after which it will infer a list of broadcasts the app may handle and a list of permissions the app requires.
3. Set up environments. Here environment means the device usage data like SMS logs, call logs, etc.
DroidBot has different policies to generate environments, for example, 
`dummy` means random environments, `static` means app-specific environments 
(static means static analysis, which gives a list of required permissions in `AndroidManifest.xml`.
DroidBot will decide which types of environments to set up according to the permissions, 
for example, if the app requires `ACCESS_FINE_LOCATION` permission, DroidBot will set up a `GPSEnv`, 
which is a stream of simulated GPS locations). 
The policy to use to generate environments is specified by the option `-env`.
4. Send user events. The events include gestures, broadcasts, key presses, etc., 
just like the events generated when a user uses the device. 
Same as setting up environments, DroidBot have multiple policy choices of sending events. For example, 
`static` policy is to send app-specific events according to static analysis, and `dynamic` policy 
is an extension of `static` which improves UI event efficient by dynamically monitoring the UI states. 
For example, 
`-event` option determine the policy to use.

Currently, to get a higher coverage, the policies we recommend are the `static` env policy and 
the `dynamic` event policy.
`static` env policy generates a minimum set of environments related to the target app.
`dynamic` event policy are more likely to trigger the most sensitive behaviours 
according to our [comparisons](#list-of-posts).

In dockerized DroidBot, we used the `none` env policy, and `dynamic` event policy.
We didn't use `static` env policy because we used a snapshot image to start the emulator in docker, 
which already had some preset environments.

## Installation
You can install DroidBot from source or run it from docker.

### From source

Installing DroidBot from source requires:

1. `Python` version `2.7`
2. `Android SDK`, make sure that `platform_tools` and `tools` are added to PATH
3. `androidviewclient`, install with `easy_install --upgrade androidviewclient`, 
or refer to its [wiki](https://github.com/dtmilano/AndroidViewClient/wiki)
4. (Optional) `DroidBox` version `4.1.1`, 
you can download it from [here](http://droidbox.googlecode.com/files/DroidBox411RC.tar.gz)

After installed the requirements, clone the droidbot directory to your working directory and run `pip install`.

{% highlight bash %}
git clone https://github.com/lynnlyc/droidbot.git
pip install -e droidbot
{% endhighlight %}

### From docker

Running DroidBot from docker is much easier.

To get the docker image:

{% highlight bash %}
docker pull honeynet/droidbot
{% endhighlight %}

or, if you prefer, build your own from the GitHub repo:

{% highlight bash %}
git clone https://github.com/lynnlyc/droidbot.git
docker build -t honeynet/droidbot droidbot
{% endhighlight %}

That's it! Refer to [Usage with docker](#usage-with-docker) section for how to use.

## Usage

### Usage with docker
Prepare the environment on your host by creating a folder to be shared with the DroidBot Docker container. 
The folder will be used to load samples to be analyzed in DroidBot, 
and also to store output results from DroidBot analysis.

{% highlight bash %}
mkdir -p ~/mobileSamples/out
{% endhighlight %}

To run the analysis, copy your sample to the folder you created above, 
then start the container; you will find results in the "out" subfolder.

{% highlight bash %}
cp mySample.apk ~/mobileSamples/
docker run -it --rm -v ~/mobileSamples:/samples:ro -v ~/mobileSamples/out:/samples/out honeynet/droidbot /samples/mySample.apk
ls ~/mobileSamples/out
{% endhighlight %}

### Usage as a command-line tool
If you install DroidBot via pip, you will be able to invoke DroidBot from command line. Try:

{% highlight bash %}
droidbot -h
{% endhighlight %}

It will print a list of options, 
in which the `-env` and `-event` are the policies of setting up environment and sending events.
and `-duration`, `-interval` and `-count` specify the how long, how fast, and how many the events are sent.

### Usage as python packages
Except from the exerciser, DroidBot integrates some useful utilities of debugging and testing.
By importing the droidbot package to your project, you will have access to the utilities.

For example, initialize DroidBot by:

{% highlight python %}
from droidbot.droidbot import DroidBot
droidbot = DroidBot(...)
{% endhighlight %}

After initialization, you can do something interesting, such as:

examining if an app is in foreground by
{% highlight python %}
droidbot.device.is_foreground('com.android.settings')
{% endhighlight %}
, adding a contact to device by 
{% highlight python %}
droidbot.device.add_contact({"name":"Alice", "phone":"12345"})
{% endhighlight %}
, simulating a incoming SMS by
{% highlight python %}
droidbot.device.receive_sms("12345", "Hello World")
{% endhighlight %}
and so on.

## Testing

### Unit Tests
The unit test scripts are in `droidbot/tests` folder. Run them with:

{% highlight bash %}
python -m unittest discover droidbot/tests
{% endhighlight %}

note that running the tests requires an emulator with serial-no `emulator-5554` already started.

### Comparisons with Monkey
You can also compare DroidBot with different policies and `adb monkey` by running the evaluator:

{% highlight bash %}
python DroidBoxEvaluator.py -h
{% endhighlight %}

this evaluator requires a DroidBox emulator already started. 
The evaluator will generate a markdown report containing the detailed comparisons. 
I have already published some of the visualized reports in [List of Posts](list-of-posts).

## Limitations

1. When running DroidBot with DroidBox, DroidBox counts accesses to sensitive resources as sensitive behaviours,
but DroidBot have to access some of them (socket, fd, etc.) to do things, so does `monkey`.
It cause DroidBox mistaking DroidBot's benign actions for sensitive behaviours (false positives).
Thus we disabled monitoring of some resources in the tailored version of DroidBox.
It is a temporary solution because it causes some false negatives.
2. Although we evaluated DroidBot by comparing the log counts of DroidBox and saw some advantages of DroidBot, 
we don't think the log count is a very reasonable matrix. 
The better matrix is `test coverage`, however we have not found a good way to measure test coverage of app.
(Most of existing test coverage tools require source code or repackaging, 
but we want DroidBot applied to all of the apps, including those without source code or cannot be repackaged.)
3. Although DroidBot is smarter than monkey, it is far less intelligent than manual testing. 
DroidBot is weak in understanding the view and dealing with unexpected situations, for example,
It may be confused by login screens, popup windows, drop-down menus, and so on.


## List of Posts

{% for post in site.posts %}
+ [{{ post.title }}]({{ site.baseurl }}{{ post.url }}) {{ post.date | date_to_string }} 
{% endfor %}