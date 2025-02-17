---
title: "Type 1 Diabetes and Observability"
date: 2025-02-16
author: "agao"
ShowToc: true
tags: ["observability"]
---

[Type 1 diabetes](https://my.clevelandclinic.org/health/diseases/21500-type-1-diabetes) is a life-long autoimmune disease where the body's immune system attacks the pancreas making it incapable of producing insulin -- a hormone needed to process glucose (sugar) for energy. As a result, insulin is required multiples a time by injection, and careful monitoring to prevent glucose levels from going too high or too low -- both of which can be fatal.

Luckily, with technology we have devices like [continuous glucose monitors](https://my.clevelandclinic.org/health/articles/continuous-glucose-monitoring-cgm) (CGMs) that can monitor glucose levels in real time without the need for finger pricks. Here I'm mostly revisiting the various solutions I've developed over the years to help me manage my Type 1 diabetes.

I've been using the Dexcom G6 for several years and it works like this: the CGM is installed on either the stomach or the arms, and there's a small thin needle that is inserted under the skin which probes and relays glucose data to your device, and optionally to Dexcom's servers.

![](/blurb/img/t1d/simple_dexcom.png#center)

However, because I use iOS there's no easy way to export this data in real-time and Dexcom's APIs have a delay (3h outside US, 1h within US) which makes them a non-option. Instead, some folks have figured out [how to get real time](https://github.com/gagebenne/pydexcom) Dexcom CGM sensor data using an auxiliary service called Dexcom Share which is used to share data with caretakers and clinics.

> If somehow you also wanted to try this out, I highly recommend checking out other more projects like [Nightscout](https://nightscout.github.io/) rather than using any of my homebrewed solutions.

## Ichor: The Very First Version

As a side project, I decided to port over the functionality from Python to Go and add some of my own flair. Ichor was a Discord bot that would display observations and other statistics from the past 12h and forecast up to 6h ahead using a LSTM model that I trained. To do this, it would render a PNG locally with the observations and predictions, and send it over a private message.

![](/blurb/img/t1d/daily_overlay.png#center)

You could also register insulin injections and carbohydrate intake, and these would be rendered and taken into account in the forecast.

I chose to use Discord because it provided a cross-platform solution that didn't require writing any frontend code since I mostly used this on mobile. There is also an official Dexcom G6 app for iOS, but I didn't like the way it looked and it didn't have all the information I needed (to do so, I would need to open another app).

### Setup

Everything was deployed using Docker and managed with Docker Compose. The Discord bot would fetch forecast projections and rendered PNGs from a separate Python container running both the model and image generation over gRPC. The biggest mistake was probably writing my own timeseries database over a key-value store like [bolt](https://github.com/boltdb/bolt), which was great for learning, and less so for actually getting work done.

## Iv2: Ichor V2

After a year or so, for some reason (that I can't remember anymore), I decided to rewrite the project with a different database (MongoDB). It supported a "real-time" dashboard-esque experience by using a dedicated channel and constantly sending and removing snapshots, and also custom alerts for hyper- and hypo-glycemia (high and low blood sugar).

![](/blurb/img/t1d/iv2_daily_overlay.png#center)

This solution worked well for a while, but it had a few problems:

- It was a hassle to update the bot everytime Discord had an update, especially as a busy college student.
- There was too much friction to run actions. I don't think the Discord API wasn't designed for that particular use case.
- It didn't work offline, or if the connection was spotty when I was out and about.

## Iv3: The Current Day Setup

Lastly, we arrive at the current day solution and it's various iterations. The goal this time was to make my life easier, and avoid management fatigue by not building bespoke code and tooling wherever possible.

> Make your predictions on how I did in the end!

I got rid of Discord in favour of using [Retool](https://retool.com/) for the dashboard and action experience and [ntfy](https://ntfy.sh/) for alerting. I was sold on the promise that Retool would be cross-platform, easy to use and would support offline mode, which wasn't exactly the case. It took a lot of fiddling initially with the components and JavaScript queries to get most of the functionality I wanted.

The biggest problem was that the graphing widgets lacked all the functionality lacked a lot of features I needed, and so I was again forced to learn Plotly to customize the graphs. And even then the offline mode was locked behind a paywall, not to mention the hours I need to refactor things to make that work.

On the other hand, ntfy worked perfectly over the years, was easy to set up and didn't require me to make any changes beyond the first hour, +1 for simple tools.

### Datadog and iOS Widgets

> Disclaimer: I work at Datadog and I like the product, but this is not an endorsement in any capacity.

After using the above setup for a while, I was getting tired of having to wait for Retool to populate the widgets with data, especially since not all of them are time-sensitive (i.e. what percentage of time was my glucose levels in target for the past week?). It was also slighly tedious to have to update the backend and then the Retool dasbhoard everytime I wanted to add a new visualization or statistic.

The solution I settled on recently was to relay the glucose data to Datadog as metrics, and use it to monitor my health. There's a few reasons to this, with the two big ones being familiarity and iOS widget support.

Since I already use Datadog in my day to day work, I was already very familiar with the query system and how to use it to get the visualizations I need. Although you could very likely also do this with other tools like Prometheus and Loki.

![](/blurb/img/t1d/dd_iv3_dashboard.png#center)

The second point however, was probably the deciding factor. Datadog's iOS app supports widgets, which allows me to add various components to my phone's homescreen (such as graphs). This has drastically improved the experience since it just takes a swipe to see all my metrics at a glance. Having it be readily available forces me to look at it more often, and stay on course.

![](/blurb/img/t1d/dd_iv3_mobile.png#center)

There is however, an issue with how frequently the widgets update. Because of Apple's [limitations](https://developer.apple.com/documentation/widgetkit/keeping-a-widget-up-to-date), the widgets aren't always updated in realtime. Instead, they are allocated a certain budget and can only update within that constraint.

> For a widget the user frequently views, a daily budget typically includes from 40 to 70 refreshes. This rate roughly translates to widget reloads every 15 to 60 minutes, but it’s common for these intervals to vary due to the many factors involved.

### Cost of Running Everything

Right now everything is hosted on a single DigitalOcean Dropet (2GB / 1CPU) which is about $13/mo after taxes. The data is periodically backed up to a Backblaze B2 bucket, which is free (I used to use DigitalOcean Spaces which was about $5/mo). I also don't pay for Retool or Datadog.

I can probably get the cost down to nearly free by selfhosting everything on our Plex server, but it's a lot of work to move everything over and set up static IP addresses. I'm not super concerned about the availability since it's not extremely critical to my health, but that is also a concern.

### Setup

For the longest time I would push a new deploy by SSH'ing into my droplet, pulling the changes from git, and then rebuilding directly on the droplet. That is until I discovered that you could just build everything on your local machine, copy the build over, and load it.

```
# Build and save.
docker compose build --parallel
docker save -o iv3.tar iv3-iv3:latest
```

```
# Copy and load.
scp iv3.tar addr:path
docker load -i iv3.tar
```

## What's Next

Recently I just switched over to using the US version of the Dexcom G6 app in hopes of being able to share my data with my endocrinologist, however not only did it not work, now I am stuck with mg/dL units rather than mmol/L. And so, I'm forced to use a third-party app ([Sweet Dreams](https://apps.apple.com/ca/app/sweet-dreams-sugar-tracker/id1644428422) for those curious) to convert and have it display on my phone and smart watch.

I'm planning on using this setup for a little bit longer before I decide on the next steps. Previously it was very much "resume-driven" and I was using it as a side-hobby to explore different things, but I think I want to be more goal-oriented this time around.
