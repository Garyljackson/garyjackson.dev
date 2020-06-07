---
title: "Adding Application Insights to your Hugo website"
summary: "How to add Application Insights to your Hugo site when using a community theme."
date: "2020-05-31"
lastmod: "2020-06-01"
author: Gary Jackson
draft: false
categories:
- Development
tags:
- "Hugo"
- "Azure"
- "Application Insights"
---

{{< admonition type=note title="Note" open=true >}}
I am only 1 week into Hugo at the time of writing - if any information here is not accurate, or there is a better way, please let me know :slightly_smiling_face::pray:
{{< /admonition >}}

## Overview
You've created your very own Hugo site and added a community sourced theme using the git submodule method detailed in the [Hugo Quick Start guide](https://gohugo.io/getting-started/quick-start/#step-3-add-a-theme).

Now, you'd like to use Application Insights to gather some metrics about it's usage. (e.g. Does anyone care about my writing :grin:)

The official (Non Hugo specific) Microsoft instructions for setting up and configuring Application Insights can be found [here](https://docs.microsoft.com/en-au/azure/azure-monitor/app/website-monitoring)

## Built in support
### Is Application Insights supported?
Or, more broadly, can I inject a block of JavaScript into the `head` html element of my Hugo website?   

As it turns out, this answer depends on whether the theme you're using has built in the option to do so.

The quickest way to find out, is to have a look at the file that defines the `head` element for your theme.


Based on the [Hugo documentation](https://gohugo.io/templates/base/#define-the-base-template), this should generally be in:  
`[the-project-folder]/themes/layouts/_default/baseof.html`

Check within the `head` element to see if you can see anything that might indicate that app insights is being loaded.   

{{< admonition type=warning title="Warning" open=true >}}
Given that the theme implementation is entirely up to the theme creator/author, what follows is a totally made-up example.  
{{< /admonition >}}

```html
<!DOCTYPE html>
<html lang="{{ .Site.LanguageCode }}">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta name="robots" content="noodp" />
        <meta http-equiv="X-UA-Compatible" content="IE=edge, chrome=1">
        <title>
            {{- block "title" . }}{{ .Site.Title }}{{ end -}}
        </title>
        {{- if .Site.Params.application_insights }}

          <script type="text/javascript">
              var sdkInstance="appInsightsSDK";window[sdkInstance]="appInsights";var aiName=window[sdkInstance],aisdk=window[aiName]||function(e){function n(e){t[e]=function(){var n=arguments;t.queue.push(function(){t[e].apply(t,n)})}}var t={config:e};t.initialize=!0;var i=document,a=window;setTimeout(function(){var n=i.createElement("script");n.src=e.url||"https://az416426.vo.msecnd.net/scripts/b/ai.2.min.js",i.getElementsByTagName("script")[0].parentNode.appendChild(n)});try{t.cookie=i.cookie}catch(e){}t.queue=[],t.version=2;for(var r=["Event","PageView","Exception","Trace","DependencyData","Metric","PageViewPerformance"];r.length;)n("track"+r.pop());n("startTrackPage"),n("stopTrackPage");var s="Track"+r[0];if(n("start"+s),n("stop"+s),n("setAuthenticatedUserContext"),n("clearAuthenticatedUserContext"),n("flush"),!(!0===e.disableExceptionTracking||e.extensionConfig&&e.extensionConfig.ApplicationInsightsAnalytics&&!0===e.extensionConfig.ApplicationInsightsAnalytics.disableExceptionTracking)){n("_"+(r="onerror"));var o=a[r];a[r]=function(e,n,i,a,s){var c=o&&o(e,n,i,a,s);return!0!==c&&t["_"+r]({message:e,url:n,lineNumber:i,columnNumber:a,error:s}),c},e.autoExceptionInstrumented=!0}return t}(
              {
                 instrumentationKey:"{{ .Site.Params.application_insights_key }}"
              }
              );window[aiName]=aisdk,aisdk.queue&&0===aisdk.queue.length&&aisdk.trackPageView({});
           </script>

	    {{- end }}

    </head>
  <body>
        Body stuff
  </body>
</html>
```


If your theme does support Application Insights, or maybe the option to inject a section of JavaScript inside the `head` element, then the contents of the `baseof.html` should give you some clue as to how you can configure it.

## Do it yourself
### Application Insights is NOT supported - what now?
This is the situation I found myself in.

Now, I know I could just clone the theme into my Hugo project and change the `baseof.html` file to include the JavaScript code to enable application insights.

I didn't want to do this because I like the idea of pulling in updates to the theme via the Git submodule.

As it turns out, Hugo will let you override any of the files in a theme, if you use the same path within your project level `layouts` folder.

For instance, in my case I only wanted to override the `baseof.html` file for the theme with my own implementation that includes the Application Insights JavaScript.

So, to override the `baseof.html` file at this path:  
`[the-project-folder]/themes/layouts/_default/baseof.html`


I created a `baseof.html` file at this path:  
`[the-project-folder]/layouts/_default/baseof.html`

My new file was a copy of the original theme version, just with the Application Insights thrown in.

I get that there is the possibility that the file could become out of sync with the actual theme file - but this still seemed like the best option to me.

Of course, I may just add the support to the official theme template and submit a pull request.


## Other thoughts
### This key seems insecure.
Wait, I'm publishing my Application Insights Instrumentation Key for the world to see? Surely that can't be a good idea.

Well, according to Microsoft, yeah, it might not be, but that's just how it is.

See the following:  
- [Microsoft Dev Blog](https://devblogs.microsoft.com/premier-developer/alternative-way-to-protect-your-application-insights-instrumentation-key-in-javascript/)  
- [GitHub Issue](https://github.com/MicrosoftDocs/azure-docs/issues/28048)


## References
- [Hugo Quick Start guide](https://gohugo.io/getting-started/quick-start/#step-3-add-a-theme)  
- [Set up Application Insigts](https://docs.microsoft.com/en-au/azure/azure-monitor/app/website-monitoring)  
- [Hugo Templates](https://gohugo.io/templates/base/#define-the-base-template)  