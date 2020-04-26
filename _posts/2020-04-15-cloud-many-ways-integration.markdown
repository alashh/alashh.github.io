---
layout: post
title:  "Cloud Integrations: One App - Many Iterations"
date:   2020-04-15 17:43:32 -0500
categories: blog integration
tags: blog azure aws integration smashlite
---
As is most of my posts it seems, a lot of these topics seem to be triggered by an interesting discussion i had with a fellow colleague. This one was related to some of the things I enjoy doing in my day-to-day. The topic in question was relating to the thing I have enjoyed the most over the years, and i'd say it has got to be taking an application and integrating it into the various cloud PaaS services at a best-practice level.

I've done bits and pieces of integration over the years, moving apps from local storage into Object Stores, Session Management into relevant Redis Caches and so on. But it has always been from the side as a small project over a week to enable a team to move faster in the cloud. What i'd like to do is distill this knowledge down into their base integrations, how i integrated and the differences of integrating say.. AWS S3 vs Azure Container Storage and what kind of changes the code take on.

To this end, I am starting this little project i'll call: *One app - Many Integrations.*

## The App
Going to be kind of simple. There's a website that is used for most fighting game tournaments known as <http://smash.gg>. The website has a lot of features but there's so much going *on* with regards to that site it runs like a dog.

So, i'd like to make a lightweight version that can display brackets for a game. They have an API available I have signed up for, so lets use this as our reason for this project.

## The Control
So, we'll need a Control to base our app off, then we can apply our Integrations one by one to find all the relevant code changes needed to actually make these apps 'Cloud Native'. Generally speaking off the top of my head for this napkin build we will start with:

- Frontend: Static Content and JS/CSS
- Cache: Session Management and API Query Caching
- Middleware: Retrieving and caching API calls from smash.gg

I think this is simple enough to start with, we can look at some kind of Account Database Auth and Database Storage later to theorize if people wanted to 'save' a certain bracket or render all brackers associated to a user.

All the above will be built in Docker Containers as well for now. So the overall flow will look something like (again, bear with the napkin here)

![useful image]({{ site.url }}/assets/cloud-many-ways-base.svg)

So, a few containers, users talking directly to the frontend (no magic, again we want our control) and a redis cache alongside a middleware server doing the heavy lifting retrieving the datasets from Smash.gg.

## Planned Integrations
From the above, I plan to add in (in no specific order)

- Load Balancers and SSL Offloading
- Cloud Hosted Caching
- Cloud Object Storage
- CDN
- Search Databases for Indexing

This will be done in both AWS and Azure, using their relevant services for each. The base logic will live in Containers for now however. I may refactor into Functions at some point but I think there is more value to be gained in just integrating (and adding) features right now.

Once the base app is wrote, and ready to go, we can start evolving it into a more 'cloud native' build application, using and integrating as much as we can from Amazon and Microsoft along the way. Through this I should be gaining a better understanding of the platforms, differences in integration points as well as the *eccentric* nature each platform has in store for me.


