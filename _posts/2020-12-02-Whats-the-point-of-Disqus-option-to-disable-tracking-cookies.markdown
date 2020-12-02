---
layout: post
title:  "What's the point of Disqus' option to disable tracking cookies?"
date:   2020-12-02 00:00:00 +0000
categories: 
excerpt: Disqus allows site administrators to prevent the embed module from storing tracking cookies on the visitors' devices. That's all good, except I don't really get the point. 
---

Disqus allows site administrators to prevent the embed module from storing tracking cookies on the visitors' devices. That's all good, except I don't really get the point.

From what I understand, this merely prevents the comments section on your web site from setting the said tracking cookie. Although if you visit some other web site that embeds Disqus and has not set this option, or even Disqus itself, a tracking cookie will still be placed on your device. The trouble is, when you come back to the web site that embeds Disqus with the tracking cookie option turned off, naturally enough, the Disqus plugin will still be able to 'see' that tracking cookie, because it is uploaded along with every data you send to Disqus (which happens when you actually comment, naturally), so you will still be effectively tracked. This is pretty shady, easy to miss out.

So basically the option just prevents you from being the one that triggers Disqus to set the tracking cookie. Although, if the cookie is set already or subsequently set, even your installation will use it more than happily. How's this a feature then, I am sure most admins do not care and simply leave the defaults on, which include tracking cookies. I mean, sure, you are safe if my site is the single one you visit, but I don't think that's even remotely the case for anyone in this world.

Recently, I have updated the site's design and naturally enough, I had to make it 'GDPR-compliant'. Or more like common sense compliant (as I prefer to have a sane approach to prompting for user consent, rather than spaming pop-ups and banners and annoying the crap out of any visitor). So, I have been looking a lot into how these services operate, so I know when it is necessary to alert the user to make a choice, and when not. When I saw the option to disable tracking in Disqus, I thought that's pretty great, problem solved, I can enable comments by default. Then I realized this.

For now, I am playing the safer bet and leaving comments display turned off by default. It also helps speed up the site loading, naturally. I am pretty sure that most users do not realize the aspects above, so even though the Disqus instance you place on your site does not create data associated with the user, it still uses that if it finds it left behind by other instances. Not pretty...

But have I got it right though? Is my judgment here correct? Ironically, you can tell me in the comments below, or simply [email me](mailto:valentingabrielradu@gmail.com) with your thoughts if allowing Disqus is not your thing. By the way, I have enabled commenting without an account in the Disqus form, so that there are fewer artificial barriers to getting in touch.