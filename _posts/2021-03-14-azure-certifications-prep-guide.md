---
layout: post
title: Preparation guide for Azure certifications
tags: [azure, certifications]
img_path: /assets/img/certifications-prep
image: 
  src: /banner.png
---

As I have passed several Azure certifications, over the years I have established a routine for my studies before an exam. I will share this in this post alongside all the tips I can think of.


## List the exam objectives

It might sound obvious but the first thing to look at is the content of the exam. As exam content evolve, but not at the same pace as Azure services, it's not that obvious to known exactly what to expect.  

### The exam page: your best source of true
You can ask people who have sat the exam a few month or longer ago, but the way to have the most up-to-date objective list is to simply get it from the exam page on the MS Learn website. Search for your exam from [here](https://docs.microsoft.com/en-us/learn/certifications/browse/), and from the exam page click on "Download exam skills outline".  

Let's take an example with the Azure Developer exam, [this](https://docs.microsoft.com/en-us/learn/certifications/exams/az-204) is the page we are looking for:  
![Exam page](/01-exam-page.png)  
Then scroll down to the "Skills measured" section:  
![Skill measured section](/02-download-skills.png)  

This section contains valuable information. First if there is a schedule change on the exam content, it will be noticed here. Then there is the sections list of the exam, with a percentage range aside. It's not clear if this percentage if related to the number of questions or how each section weights in the score calculation, but you should pay attention to this percentage and use it when you prioritize your studies.  

### Making a to-do list
What I usually do from here is download the *skills outline* document, and build a to-do list from it like this:
- Create a section with each section of the exam, sorted by percentage (higher to lower)
- Within each section, write the list of the objectives
- Finally add a checkbox for each bullet point under each objective  

I use OneNote for this, create a notebook for each exam, here is an example for the AZ-202 exam:  
![OneNote](/03-onenote.png)  

Then the idea is to tick all the boxes beside the things I already have done or know enough, and leave it unchecked for the things I need to work on. 
Throughout my studying I check the boxes to track my progress. I don't necessary need to tick everything, I might skip some items if the section percentage is low or if it's something I barely know.

### Be aware of changes !
Last tip on the exam objective, remember that the exam page and the skills outline document are you only source or up-to-date information. If you plan to take your time for studying, get back to it from time to time to see if something has changed.


## Use Microsoft Learn

Listing the things to do is first step, a quite boring one, and is less important since the release of the Microsoft Learn [website](https://docs.microsoft.com/en-us/learn/). You'll find resources online to find out [what Microsoft Learn is](https://docs.microsoft.com/en-us/learn/support/faq?pivots=general), but shortly it's a platform with tons of tutorials on how to use Microsoft products, including Azure services.  

It's free, content is growing, and it's kinda fun as there are *gamification* features with levels, xp, and achievements.  
![Ms Learn XP](/04-mslearn-xp.png){: width="350" }
_See how much xp I have 😎_

I strongly encourage you to use it as one of your primary sources of learning, with the following additional tips:
- Create an account to track your progress and gain xp, it's more rewarding/addictive
- Use bookmarks and collections to save up things you wanna learn
- Under each exam page, there is a list of *learning paths* to follow to prepare for the exam. Take as much as you can !
- There are some questions after each module, which are relatively easy. Don't be too confident if you always find the good answers, as the questions is the exams are usually harder 😉


## MeasureUp practice tests

At some point you will consider getting a practice test from MeasureUp, as those are official practice tests. I have already used them a few times, here are their advantages in my opinion:
- The interface is similar, but not exactly like the real exam UI. So if it's your first exam, it helps to get familiar with the exam experience and the type of questions.
- Each answer comes with an explanation, and resources to know more about it. So if you're wrong, you can learn why and improve

![Sample question](/05-measureup-question.png){: width="650" }
_A sample question from a practice test..._

![Answer explanation](/06-measureup-explanation.png){: width="650" }
_...and the answer with explanations_

Additionally here are some tips I can provide on the practice tests:
- Do not spend too much time working on them. It's tempting to dedicate almost all your studying time to it, as you want to improve your score, but be careful. I have already done this too much, and at the end I knew almost all questions by heart, which was worthless. Remember you're preparing to get certified on a piece of tech, not on a set of questions !
- Opinions will vary on this, but personally I have almost never met a question in a real exam that was in a practice test. This a good and a bad point, the practice test does not provide the questions in advance, it prepares you to think in the way the real exam is made.

Lastly if you search for practice tests on the web, outside of the official ones you will find tons of websites that guarantee 100% success with *real* questions. Do not trust them, these are bullshit, barely legal sources I think, and the worst way to prepare for an exam: don't try to fill your brain with pre-answered questions, try to learn something that will help you in your everyday job.


## Other sources of information

As there a also great resources outside of the official ones, let me share here some of my favorites.

### Blog posts
Remember the list of objective I mention at the beginning ? Well I'm not the first with this idea, even better there are some great people who do this and share resources associated to each objective:
- Gregor Suttie, aka Azure Greg, is an MVP who shares a huge blog post each time he passes an exam, which happens a lot. Checks out [his blog](https://gregorsuttie.com/) and look for *study guides* posts
- Stanislas Quastana is a cloud solution architect for partners at Microsoft France. I had the chance to meet him a few times for training on certifications. His preparation guides on [his blog](https://stanislas.io/) are worth checking out, and packed with super great PowerPoint decks. Some of his posts are in French 🥖🐓

### Reddit
The [Azure subreddit](https://www.reddit.com/r/AZURE) is a place where people can discuss about Azure stuff. There are often discussions on certifications, whether it's a preparation guide or folks sharing their experience on exams.  
The search feature is handy, search for your exam code, limit the results to the Azure subreddit, and you might find some valuable informations (example with the [AZ-204](https://www.reddit.com/r/AZURE/search?q=az-204&restrict_sr=1) exam).  

Careful though, in the discussions there are some people (maybe bots) who advise to go to some of the shitty websites I mentioned earlier, with *dumps* and *100% guaranteed success*. Once again avoid them like the plague, you'll have more chance to infect your computer with a virus than learning something worth it using them.

A little reminder, when you're looking at a blog post or a discussion on Reddit, always check the date of the post. Exam content always change, so the fresher the post is, the better. If you have any doubt, check on the exam page !

### Pluralsight (and other learning platforms)
I don't use it very much as I prefer reading text over watching videos, but Pluralsight has some great resources to prepare for an exam. I heard that Pete Gallagher provides great [courses](https://app.pluralsight.com/profile/author/peter-gallagher) for those who are preparing for the AZ-220 exam on Azure IoT for instance.  
One feature I particularly like on Pluralsight is the [Skill IQ](https://app.pluralsight.com/skilliq) assessments. For each topic of an exam, it provides a set of questions, with limited time to answer, and at the end you get a score and how you rank among other "students". Similarly to the practice tests, it's not the best feature for learning, but it helps to gain confidence.  
There are other learning platforms like Udemy or Linux Academy that I have slightly used, you can go check them as another source of learning content. 


## Wrapping up

I'm done writing about Azure certifications for now, I hope this post and the previous one will help if you want to get certified. If that's the case, great, remember that you're doing it for yourself and to learn something new.  
And if you don't want to, it's fine, don't feel forced to get certified, you can have a happy professional life without that.  
Once again, happy learning ! 🤓