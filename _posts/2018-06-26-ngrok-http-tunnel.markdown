---
layout: post
title:  "Http tunnel for debug through Ngrok"
date:   2018-06-26 23:30:54 +0300
categories: [Tools]
tags: [Tools]
---

Hi! I'd like to share with you some tool that I am using.

Sometimes I need to debug set of applicaions that are talking to each other throught http/https protocol. For example mobile application and server application.

Problem with debug is establishing connection from mobile app through internet to server that is running at localhost environment.

I found a tool that helps me with it a lot and allow to establish connection in one minute: it is [https://ngrok.com/](https://ngrok.com/) This terminal application has powerfull abilities for different cases (just see [docs](https://ngrok.com/docs)).

For example I am running it in a next way: `ngrok http 60370 -host-header="localhost:60370"` where 60370 is a port of web server application. 

Application will provide you with a https link that you could use for connection to it. I am using that link in a mobile app as a server url and it just works perfectly.

Get lucky !