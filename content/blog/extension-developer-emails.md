+++
title = "The strange, sketchy emails an extension developer receives"
description = "Where does extension malware come from? It might start from emails like these."
date = 2021-07-09
[taxonomies]
tags = ["chrome-extensions"]
+++
If you've followed tech security news in the past year, you may have heard about the widespread propagation of malware in browser extensions. In December of 2020, it was reported that up to 3 million devices had been affected by malware identified in 28 Chrome and Edge extensions[^1]. As ArsTechnica reports, this has been a recurring problem over the past few years due to the incredible power granted to browser extensions and the poor vetting process that allows malware into official extension stores.

So where does all this malware come from? Some may come from extensions that are developed malevolently from the beginning, or from legitimate developers whose source is compromised. But from my experience, I believe that some must come from extension developers who willingly place suspect tracking code in exchange for payments from illegitimate "advertising" companies.

As the developer of [a small Chrome extension](https://chrome.google.com/webstore/detail/search-bar-for-classroom/dmlfplbdckbemkkhkojekbagnpldghnc?hl=en) that has had up to 20,000 users, I saw a side of this problem that is not visible to the majority of those interested in this problem. After publishing the extension last year, I received multiple requests to either sell the extension or partner with an advertising firm, which would invariably involve the addition tracking code into the extension's source, with or without the user's consent. 

This is particularly dangerous in the case of browser extensions since Chrome automatically installs updates to extensions as soon as they pass the automated review, meaning that initially benign extensions can suddenly become [Trojan horses](https://en.wikipedia.org/wiki/Trojan_horse_(computing)).

Now keep in mind, my extension simply adds a content search bar to the free educational software Google Classroom, so I would never consider monetizing it, not to mention doing so with intrusive ads. Nonetheless, as a result of being on the Chrome Web Store, I was hit with what I assume are dragnet email requests to developers. The first of these emails, which I received from the name "Валерий Земов" in October of 2020, is perhaps the most striking. Here is the body in full:

> Subject: extensions buyer
>
> Hello
>
> My name is Valerie.
> Our team is engaged in the placement of native ads using browser extensions for this.
> 
> The most important thing for us in our work is the ability to quickly increase the volume of our traffic. We know that developing our own extensions takes a lot of time and effort, so this way is not suitable for our business.
> 
> Important part of our job is buying browser extensions. For this I am writing to you.
>
> Yes, we are ready to buy your extension «Search Bar for Classroom», and I am writing to you to find out if you have a desire to sell it?  We are ready to pay you from 0.2 $ for each of your users. We are ready to offer you 4.000 $. I am open to dialogue and will wait for any of your answers.
> 
> I will be happy to receive any answer from you. Perhaps if you don't want to sell your extension you know someone who wants to do it.
> 
> Thanks for your time anyway!

This is incredibly interesting since immediately we see how much this individual, or the company they represent, value the indiscriminate access to a user's browser extension: 2¢. It makes me wonder what they would have done with my extension to make back that investment, but if I had to guess, it probably wouldn't have been completely benign.

The next request came from "serg scream" on January 27, 2021. This time, instead of an extension buyer, this person wants me to insert their code into my extension, which will seemingly add additional advertising to Google Search. Again, here's the email in full:

> Subject: Hello. I'd like to offer a new ad format for your consideration.
>
> ABOUT US
>
> We are a young modern team of specialists engaged in developing products for monetization of browser extensions. Our new product called uSearch is a groundbreaking promotion system for sales on the sites. The key principle of operation is that in search engines partners' extention will display the advertisers' icon left to the page name thus attracting attention of users.
>
> OUR ADVANTAGES
>
> \* New product on the market, with high profitability for extension owners
>
> \* uSearch product does not cause any negative emotions in users because it simply highlights partners in search results using their shop icon
>
> \* uSearch is an absolutely legal system that's why we do not face any conflicts with search engines because our product fully complies with the rules.
>
> \* Weekly, automatically-performed payments
>
> \* We are always in touch. Any of your questions will be quickly resolved via Skype or Telegram
>
> HOW DOES IT WORK?
>
> 1\. You place our code in extensions
>
> 2\. During the search users will see highlighted icons of our partners' shops.
>
> 3\. After purchasing something on the partner's shop website you will receive your percentage of the conversion
>
> If interested, we can show examples how it works when our code has been installed.

This one is an absolute goldmine. First of all, I love how they assure me that their product "does not cause any negative emotions" and is "absolutely legal," I think that inspires a lot of confidence. More seriously, this is exactly the kind of request that could lead to the installment of a Trojan horse extension. Of course, they claim that all their extension does is add some advertising to their search requests (if that is not intrusive enough), but I imagine they wouldn't mind including some additional code to scrape user data while they are at it.

One more, though this one may not be as exciting since it is certainly the most legitimate-looking of the three (admittedly, not a high bar to clear). This one comes from a company called [AdExpertsMedia](https://www.adexpertsmedia.com), which at least from a first glance does not seem to be too sketchy. Here was their email:

> Subject: AdExpertsMedia | Search Syndication Possible Cooperation
>
> Hey Search Bar for Classroom team!
>
> Our company specializes in all aspects of downloadable software distribution & monetization, search syndication.
> 
> I would like to check the possibility of cooperation between our companies.
>
> We have best performing search feeds for search syndication for mobile&desktop traffic.
> 
> Best regards,

Now, I don't know exactly what this company intended to add to my extension, nor whether they had any malicious intent beyond targeted advertising, but what worries me about requests like these is the unique power of code that runs in Chrome extensions that can lead to tracking that is even more extreme and intrusive than the techniques found in a modern ad network running on a website. 

Beyond just the cookies that can be used to track users across the web, Chrome extensions offer many more ways to mine a user's browsing data or personal information, as can be seen in the numerous malware-ridden extensions discussed previously. Some of these abilities are locked behind prompts for user permissions, but especially if they come from an extension that has already been installed, users are likely to just blindly accept an extension's requests for further permissions.

Needless to say, I did not respond to any of these requests, though in hindsight, it might have been fun to probe these senders for more information about their businesses, and how exactly they planned to monetize my users. It does make me wonder how many other extension developers may have received these kinds of requests and 

I hope you found this post interesting, and as always, please feel free to reach out [via email](mailto:micahcantor01@gmail.com) with any questions or comments.

### Notes

[^1]: https://arstechnica.com/information-technology/2020/12/up-to-3-million-devices-infected-by-malware-laced-chrome-and-edge-add-ons/