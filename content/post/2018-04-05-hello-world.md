---
title: "Hello World"
subtitle: ""
date: 2018-04-05T17:00:00-05:00
tags: []
---

Back in the before times (aka the mid-00's), I used to have my own site. Xanga, MySpace, & Facebook were the hip places to be, but as a person who liked technology, I wanted to be cool and host my own content. Enter iWeb<a href="#note1" id="note1ref"><sup>1</sup></a>. [iWeb](https://en.wikipedia.org/wiki/IWeb) allowed me to create pages without mucking around with raw HTML too much and had built in FTP support so I could push to my server hosted by GoDaddy. 

After a while though, I couldn't justify the hosting cost (pre-AWS era, things were expensive). Traffic was always pretty low and then Apple discontinued iWeb. At that point, I decided to shut down the site and just use Twitter and Facebook.

Fast-forward to today where blogging/micro-blogging sites are everywhere. Rather than use Medium<a href="#note2" id="note2ref"><sup>2</sup></a> or WordPress<a href="#note3" id="note3ref"><sup>3</sup></a>, I wanted to use something simpler, but still have some sort of control over how my content was hosted. I looked at [Ghost](https://ghost.org/) and it seemed cool, but was too expensive for a small non-commercial site to use their hosting. Then, I started looking at other static site generators and ended up with two: [Jekyll](https://jekyllrb.com/) and [Hugo](https://gohugo.io/). GitHub Pages uses Jekyll, so it has a very robust support community and is battle tested. However, Jekyll isn't a singular package as it is a combination of a lot of gems (not to mention the additional stuff for [GitHub Pages](https://github.com/github/pages-gem)). That's where Hugo shines as it is written in [Go](https://golang.org/) so it is distributed as a single binary. Go also makes it super fast. Now, you may be wondering why it being fast matters... the reason is is that I'm writing this on my 2008 MacBook Pro. Ruby's overhead isn't that bad, but on a machine that only has an Intel® Core™2 Duo ([T9600](https://ark.intel.com/products/35563/Intel-Core2-Duo-Processor-T9600-6M-Cache-2_80-GHz-1066-MHz-FSB)), I need all the help I can get.

Getting started was quite simple:

1. Install [Homebrew](https://brew.sh/).
2. `brew install hugo`
3. `hugo new site blog`

Once the site was created on my machine, I browsed the [theme repository](https://themes.gohugo.io/) and found one I liked. Adding it was simple too: all you have to do is add a git submodule in the themes directory and update the site config to reference the theme name and you're good to go.

All in all, I'm pretty happy with how things have turned out and how nice it is to just write in markdown and not have to worry about any HTML/CSS/JS shenanigans.

<a id="note1" href="#note1ref"><sup>1</sup></a> 'Member iWeb? I 'member. <br>
<a id="note2" href="#note2ref"><sup>2</sup></a> Medium is a bit shady since they are marketing your content to generate income for themselves and you can't customize the styling of your content. Also, [Dickbars](https://daringfireball.net/2017/06/medium_dickbars). <br>
<a id="note3" href="#note3ref"><sup>3</sup></a> Security [nightmare](https://www.cvedetails.com/vulnerability-list/vendor_id-2337/product_id-4096/).
