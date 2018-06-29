---
layout: post
title:  "Experimenting with Data Explorers"
date:   2018-06-01 00:00:01 -0000
---

Interactive data explorers can help refine questions for better collection and suggest new areas of inquiry.  Data explorers should present filtered sets from the universe of data that reveal patterns in expected behavior along a measured dimension of analysis.  Patterns are often generated from behavior tracking installed in mobile applications.  They can also incorporate predictions modeled from public data.  No matter what the source is, carefully defining the graph's axes and filtering mechanisms can make an explorer's operation intuitive, even in a complex tracking system.

<!--break-->

The first piece of software I wrote professionally was a data explorer built for the Philadelphia Workforce Investment Board on behalf of the Philadelphia School District in 2008.  The data explorer was written in [Processing](http://processing.org), a Java-based library that simplifies programming into visual concepts.  The explorer compared the vocational programs found in the school systems to the number of jobs projected to be available for graduates over a ten year time span.  The explorer could be modified on several dimensions including subject matter, geography, and school.

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/engineer/2009_PWIB/3.jpg" width="640" alt="Career and Technical Education Explorer">

In the image above, two Philadelphia high schools are compared to see which school prepares students for the highest wages inside of the city.

Behavior tracking is common in almost all iOS applications.  In addition to attaching common tracking systems to the purchase flow in [Beautified](https://voxels.github.io/professional_experience_0606018#beautified), I also sent raw event information from [mParticle](https://www.mparticle.com) to an AWS bucket.  The graphs below are screenshots from an interactive data explorer built in Processing that reveal the step a typical user might end their session before getting to the point of sale.  

Every user session in the collection is presented on a time based axis that flows around the origin.  Each step in the purchase flow is presented in a different color, at a different radius, so that a trip from the outer circle to the inner circle shows how far a single user penetrated into discovering and then buying a product.

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/researcher/behavior_tracking/2014_UserAcquisitionAndBehavior_Legend.png" width="640" alt="User Acquisition and Behavior Legend">

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/researcher/behavior_tracking/2014_UserAcquisitionAndBehavior_Filter.png" width="640" alt="User Acquisition and Behavior Filtered">

The product owners could use the explorer to develop a sense of how well the design was pushing users across the purchase flow over the course of a single session.

Data explorers can not only be useful but also graceful.  In the explorer below, I assigned characteristics from a spreadsheet of contacts to [soft-body](https://www.amazon.com/gp/product/0985930802/ref=as_li_tf_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0985930802&linkCode=as2&tag=natureofcode-20) agents.  Filtering through the various characteristics drives a reorganization of the agents into new groups, so that an overall sense of belonging can be discovered.  

{% include youtubePlayer.html id="KP4pgTmJO-U" %}

The fluidity of the behavior lends some personality to the agents, and self-organization of flocking systems is a trait that might be further refined.  Since we recognize relatively motion almost as well as we see color, position, and size, agent-based behavior might be an untapped resource for mixed reality visualization.  Every so often, I expect to come back to this topic to see where technology and art can produce a more natural interface for data exploration.

