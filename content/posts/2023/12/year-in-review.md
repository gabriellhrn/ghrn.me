---
title: "Year in Review: 2023"
date: 2023-12-26T10:45:00-03:00
categories: ["year in review"]
tags: ["aws", "blogging", "career", "cloudflare", "coffee", "hiking", "hobbies", "meetup", "talks", "travelling", "writing"]
---

One of my goals for this year was to write 3 blog posts. This is my third
post. One could argue that a year in review kind of content is cheating, just
something to beat the goal I set; or that I'm following the trend, after all everyone and
everything is doing a 2023 retrospective now. And I could agree with that. It is
partly true. But it is also true that I've done it anyway, in my own
mind, remembering the nice things that happened and the things I set out to do
but didn't. And I think it's a good thing to remember your year. Just like we do
at the end of our Scrum Sprints: think about what went well and you want to do more
of, but also what was not good so that you can improve on the bad things.

This is the first time I recall my year publicly like this and I can't start
this review without first mentioning this <!-- vale proselint.Very = NO -->very<!-- vale proselint.Very = YES --> blog.

## My blog

I wouldn't be able to reach my goal of 3 blog posts this year if I had not first created a
blog. Well, technically I could. I could use a blogging platform. There are tons
of them. But I wanted my own thing. So I went with [Hugo](https://gohugo.io/),
found a neat minimalist [theme](https://github.com/g-hw/hugo-theme-nostyleplease),
made a few tweaks, and published my posts. Creating the blog and writing the first post,
even though feeling they were not perfect, was the best thing I did. This is
something I've learned from my colleagues in Product and that I keep reminding me
of: it doesn't matter that you've implemented an impeccable piece of engineering if
no one is using it. Scoping tasks, prioritising and defining a roadmap are
skills that I don't master, but I feel I've improved on them this year through
of projects I've led at work. But I digress, don't I? I'm supposed to be talking about my blog. Shall we get back to it?

Although I haven't published many posts, I've received good feedback on the few
ones I wrote. I don't really know how many people read the blog, because it has no
analytics. But that's fine with me. So far, the hardest part of blogging has
been coming up with ideas for posts, and starting to write. I find writing enjoyable, but
starting to write is difficult. When I do start, it gets easier. My goal for
next year is to do twice as much as I did this year and write 6 posts.

## New role at work

The tech sector now is different from what it was a couple of years
ago. Employees had a lot of power to choose between companies and there were
plenty of opportunities. But they rared. Layoffs and economic instability have
changed how we see tech now. Talking to friends, a common thing I hear is that
they don't feel like their jobs are secure anymore. As a result,
combined with a company restructuring, my expectations for promotions were at a
low ebb.

I joined Delivery Hero as a Senior Systems Engineer, working in the Platform
domain of the Fintech vertical. Since joining my new team, I wanted to deliver projects with
high impact and be influential. I didn't just want to work on my tasks; I wanted
to lead initiatives and strengthen my project, product and software engineering
skills. I got rejected for promotion in the first performance cycle I was
eligible for. It never feels good getting rejected, but, trying to understand
the reasons, I reached out to my manager, his manager and his manager's manager.
I wanted to hear the perspectives and expectations of people with different
scopes and levels of influence. And so I truly listened to their feedback,
focused on the areas where I needed to improve, and on the next cycle the
result was different: I got promoted, despite my worries about the market.
By all means, this is not the full story of what happened. I want
to write about my process of growing to Staff Engineer, what my
expectations were and how it's been so far. I feel like this is the first time
I didn't get a promotion or raise naturally; I had to focus on this goal, while
humbly and constantly listening to feedback and adapting. One of my posts next
year will be about the transition to this new role.

## Talk at Cloudflare Peer Point Berlin

There was a message on my company's Slack one day from our Cloudflare's Customer Success
Manager. She asked if anyone in was interested in presenting at
Cloudflare Peer Point Berlin, a meetup organised by Cloudflare. My proposal was
to talk about how we used Cloudflare in my vertical, mention some projects and
use cases, and also talk about future plans, what new features we wanted to try
and why. I ended up talking about Cloudflare Workers, Transform Rules, Tunnel and
mentioned API Gateway and Zone Versioning.

There were two great talks before mine: [David Tofan](https://davidtofan.com/)
talked about recent Cloudflare announcements and
[Mark Dembo](https://de.linkedin.com/in/mdembo) gave a cool demo about building a
streaming site with Cloudflare Pages and Stream. All in all, the event was fun
and well organised. The room of about 100 people made me nervous in the
beginning, but as I started talking I got more comfortable. It was a good
experience that I would like to repeat.

## AWS re:Invent

This year I attended my first ever re:Invent, the massive conference organised
by AWS in Las Vegas. It was my first time in the city too. I was excited about the
event, selected the talks I was most interested in weeks in advance and
prepared my agenda. However, the long trip from Europe, the lack of sleep and the
dry climate got the best of me and I fell ill. For two days, I was extremely
tired, had body aches, a cough and a runny nose. I got better, but continued
feeling tired and distracted. Still, I tried to enjoy the event at least a little.

My favourite talk was [Surviving overloads: How Amazon Prime Day avoids
congestion collapse (NET402)](https://www.youtube.com/watch?v=fOYOvp6X10g).
Jim Raskind is a great speaker. His part of the talk was engaging and full of insight.
He talked about how overloaded systems build up queues of requests to process and
become irresponsive, even though they're
utilizing their resources fully. For instance, a system is dealing with more
requests than it can process, the queue of requests starts to increase, users or
other systems start to retry because they waited for too long, and their new requests are also
added to the queue. However, their old requests have not been cancelled and get
processed, but the response will never be seen because the client retried.
So the system ends up busy processing stale requests while new requests
wait in the queue and timeout.

He called this the spiral of death. Three advices he gave were to test your
systems to failure and beyond; fail fast is good;
and look for problematic retries. Ankit Chadha then
ended the talk by recommending AWS tools that can be used to detect congestion,
mentioning the importance of metrics and alerts, and explaining
how some AWS services can be used to prevent malicious attackers from putting your
systems into a state of congestion collapse. I highly recommend watching the
talk.

I also enjoyed the talk [Modernization of Nintendo eShop: Microservice and platform engineering (GAM306)](https://www.youtube.com/watch?v=grdawJ3icdA).
In this talk, Shinya Ogura told the story of the Nintendo eShop, how it moved from
on-premises, to AWS and finally how it was modernised to effectively use the
capabilities of the cloud. He also mentioned their decision to build an
internal developer platform without dedicated platform engineers. Instead, they
brought together their DevOps Engineers and Software Engineers in a working
group to address the pain points of delivering their software into production.
This was also an insightful talk that I recommend watching, especially if you're
building an internal developer platform or if you're managing teams responsible for
it. It doesn't hurt to say that you can learn from this presentation, but take it with a
grain of salt: what works for Nintendo may not work for you.

<!-- vale proselint.Skunked = NO -->
I haven't mentioned the keynotes in detail, but they were also really good and
well worth a watch, especially Werner Vogels' talk. To conclude, although I felt sort of
overwhelmed by the conference, the overall experience was positive and I'm
looking forward to going back to it, hopefully in a better health this time.
<!-- vale proselint.Skunked = YES -->

## The non-work related stuff

Moving on from work related topics to more personal things, two hobbies I have
are fitness and travelling. The year of 2023 was a year
of hiking trips for me. In August I travelled to Bergen, a beautiful city
surrounded by mountains and fjords. The view from the top of the mountains was
amazing. The climb was also fun. I didn't choose the right shoes for the hike
and I ended up with some nasty blisters. But you wanna know the truth? It was
worth it. I also went on a boat trip through the fjords, which lived up to my
expectations: the fjords are really beautiful.

Photos don't capture the whole experience of being there, enjoying the view at
the top after a difficult climb, but still they describe the scenery better than
my words. You can see some of the photos I took in Bergen
[here](https://photos.app.goo.gl/HR9Gv4pov9MU84Vs9).

The other hiking trip I did was to Pirna, to hike the Malerweg. My
girlfriend's parents came to Germany just to hike this trail. Oh, and to visit
their daughters too, of course. I hiked two stages of the Malerweg and they were
definitely more challenging than my hikes in Bergen. Longer distance, more
changes in elevation, slippery stones. But the views were great and the
hikes were fun. Some [photos](https://photos.app.goo.gl/yWu9sqbyUkDoQQ8t5).

Another hobby of mine is coffee. My collection of coffee equipment is not small, I
have to be honest. Maybe I could write a dedicated post just to talk about my
coffee equipment, but this mention here will be brief. This year I upgraded my
grinder from a Timemore Chestnut C2 to a Comandante C40 MK4. My Timemore was
already beat up from usage. It was getting harder and harder to grind with it, but the
poor thing had been used heavily for close to 2 years. It was good value for
money. The Comandante is better though. It feels more robust, sturdier. I'm
happy with the upgrade. My collection of drippers has also grown. My
birthday present from my girlfriend was an Origami, a dripper I've been eyeing
for quite some time. It's a beautiful piece. It is more complicated to handle than
my V60 or my Kalita, but I like the coffee it makes. And did I mention that
the Origami is really pretty?

## Looking ahead

With so much going on in our daily lives, it's easy to forget what's happened
<!-- vale Vale.Repetition = NO -->
over the year. Now, I'm not one to dwell on the past. What happened
happened and there's no point in thinking about how it could have been
<!-- vale Vale.Repetition = YES -->
different. But by reflecting we can learn what we can do better or what gives us
energy. This year I wanted to read more books, I wanted to finish some courses,
I wanted to finish some projects that were important to me at work. But I couldn't, for one
reason or another. These are things I set out to do, but didn't. Some of these
were by my own fault, others by change of priorities. So now I have to set my
goals for 2024, looking back at my failed goals and asking myself if they still
make sense.

And with that, I close my 2023. Now to set my goals for 2024.
