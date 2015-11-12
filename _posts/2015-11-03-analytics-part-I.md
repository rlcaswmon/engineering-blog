---
layout: post
title: "Anatomy of Analytics"
subtitle: "Part I"
description: "As use of Analytics is becoming more pervasive at Monsanto, this series of posts will discribe the supporting architecturet that is being used to instantiate a complete analytics platform"
header-img: "img/mon-maui.jpg"
authors:
    -
        name: "Naghman Waheed"
        githubProfile : "nnwahe"
        twitterHandle : "nnwahe"
        avatarUrl : "https://avatars1.githubusercontent.com/u/12257134?v=3"
tags: [analytics, architecture, platform, information, business intelligence]
---
Years ago, when I made the switch from procedural to object oriented 
programming, concepts such as function overloading fascinated me. 
A single function changing and behaving based on the types of values 
passed to it spoke volumes to its simplicity and elegance. Feeling 
a bit nostalgic I am going to say “those were the good old days!". 
Today, the word ***Analytics*** is certainly an overloaded term
in the data world!  It is used to define work done by data scientists 
to business analysts and from execution of sophisticated statistical 
algorithms to the use of simple visualization tools. Oh, and let's 
not forget its synonymous use with the term “Big Data”. I wish I
could muster up the same sense of euphoria for the overloaded term 
***Analytics*** that I once felt for the use of overloaded function
in a class definition. 

In this multi-part series of posts I will share practical knowledge 
about using data to gain insights that I have acquired during my 
career as a data and information architect. In addition, I will 
highlight the discipline that is required to create the supporting 
architecture and challenges encountered along the way when instantiating 
a comprehensive analytical platform. The first step on this journey is 
simply defining the consumers of information and understanding what they 
need. In short, the first commandment is simply - know thy information 
consumer. Figure 1 captures the essence of this concept by defining four
distinct types of consumers of data - information consumer, information 
pro-consumer, data scientist and machine. 

![Information Consumer Types](/img/Information_Consumers.png)
<center> Figure 1 - Information Consumer Types </center>

One thing to note in Figure 1 is that the word ***Analytics*** is sparingly 
used. This was deliberate. All information presented has analytical 
value. More on that in the next post. For now, let’s just say before 
the word ***Analytics*** took on a life of its own, many of the terms above 
sufficed to describe how different consumers analyzed data and information. 
It was not long ago that the terms such as “slicing and dicing” and 
“drill down analysis” were commonly used to describe capabilities which 
allowed users to make important decisions. Today those capabilities are
still an integral component of Business Intelligence and offered by 
many vendors as a standard offering as part of their respective suite of 
products. 

A few observations on information consumers in each category:

***Information Consumers*** are generally the business users and executives 
that want access to timely and accurate reports and dashboards through 
intuitive and user friendly interfaces. The information consumed by 
this group of consumers needs to be highly accurate, trusted and 
reproducible.  Additional characteristics of information consumed by 
this set of users could be simple aggregated views of past sales or 
financial data for a specific region, as an example, or a view of 
forecasted sales numbers generated thru some sophisticated prediction 
algorithm.

***Information Pro-Consumers*** are commonly known by more familiar terms 
such as business analysts and power users. This group is concerned with
performing analysis on data and generally their focus is to understand 
the past in order to predict the future. In addition to accuracy and 
timeliness this group often demands low latency and high accessibility
to data. More over, this group prefers to have multiple tools at their 
disposal to be able to perform aggregations, calculations, slicing and 
dicing as well as adhoc analysis on data. 

***Data Scientists*** do not need any introduction. Lately, much has been 
said and written about this set of users. A quick search on the web will 
prove that. But this series is not about defining who they are. Instead 
it is about what they would like to do with data. Regardless of whether 
these users are statisticians with computer science degrees or computational 
experts in some specific field, their needs revolve around having access 
to any and every piece of data - big or small, structured or unstructured. 
Often the discovery process and experimentation undertaken by this group 
may include diverse techniques ranging from simple data mining to
applying various statistical algorithms to testing some hypothesis. 
In the end applying sophisticated statistical techniques and/or machine 
learning algorithms to create insights is the primary motivation behind 
the work of this user community. A few posts from one of my colleague are
an interesting read on this particular data consumer type.<br/>
  [What makes a data scientist – Part 1][Scientist1]<br/>
  [What makes a data scientist – Part 2][Scientist2]
[Scientist1]: http://engineering.monsanto.com/2015/11/09/what-makes-a-data-scientist-part-1/
[Scientist2]: http://engineering.monsanto.com/2015/11/11/what-makes-a-data-scientist-part-2/

***Machines*** operate on data and need data to operate. Despite this fact, in 
the data world, the needs of this voiceless group is often overlooked. 
With increased mechanization and automation, system-to-system communication 
is only increasing in importance and scale. Whether it is data collected 
through sensors and passed on to other systems for processing or data made 
available via APIs for general consumption by external users or partners, 
the fact remains that most companies are still trying to shore up their 
ability to service this important category of consumers. 

In conclusion, the message is simple. It is critical to understand the need 
of those you serve. Such basic yet crucial piece of information should be a 
precursor to everything else that follows. Whether it is creating a reference 
architecture and/or standing up a scalable analytical platform one thing is 
immediately apparent from figure 1.; the platform will need to cover use cases 
ranging from machine learning to ad-hoc analysis, data analysis to sophisticated 
analytics and from simple information consumption thru APIs to machine to 
machine communication. A key mandate for a successful analytical platform is
to make sure that the need of each consumer type is addressed holistically 
in helping them turn data into useful information and insights. 


