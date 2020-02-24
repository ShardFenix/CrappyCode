# CrappyCode
This is a collection of programming horror stories that I have encountered over the years. Most of these were found as I was working at an insurance company. They are face-palmy enough that I decided to share them. Perhaps you can learn some lessons from these stories, just as I have.

### The Case of the Uncommitted, Untested, Unoptimized Code
We owned a batch job that ran daily. Until this incident, we'd never had a problem with it. One day, it started throwing OutOfMemory Exceptions - likely due to our rapidly growing customer base. I looked at the stack trace to find the line it was blowing up on (as you do), and I opened up my Subversion client to check out the code.

When I opened the file to see where the exception was being thrown, I immediately knew this was going to take longer than expected, because the line throwing the exception was blank - it was whitespace. This means one and only one thing - the code from SVN was not the code that was in production. Apparently, the dingus who wrote this job didn't believe in version control or deployment pipelines, and the last time it was built, he built it from his local machine but never checked the code back in.

So began a decompiling spree. Luckily this batch job was relatively simple, so I only had about 15 java files to recreate and fix. Once I rebuilt the project from production bytecode, I tried finding out how he tested it. I asked some of my teammates if they knew how it was tested, and apparently the snowflake who wrote it didn't believe in having a test environment. He tested the thing in production. So, I had to add some config features to allow switching between environments.

Once I finally got the thing running against our test system, I learned that this thing was pulling hundreds of thousands of database rows into memory during its initial query. This job is supposed to grab data from one database, do some transformations, and drop it into another database. The first step is where the OOM was happening. I ended up talking to someone who better understood the tables this batch job was reading from. Together, we learned that this query was doing not one, not two, not three, but FOUR redundant table joins. This means the result set it was extracting was 4 orders of magnitude larger than it needed to be, entirely filled with duplicate rows. Apparently the translucent jellyfish who wrote this thing didn't understand how relational databases work or how to write sql queries. This was also the reason the batch job (which runs every day) was taking 18 hours to finish, until it stopped working.

Once we fixed the query, we ran it. It finished in 30 seconds.

### Beware of version upgrades
We had an initiative at the company to keep as many similar applications on the same tech stack as possible (eg all webservices would use the same container and framework). We were migrating from one container to another, and in the process we had to update some versions of Spring and Hibernate. Some of the database access code in this particular project only worked on the older version - when we upgraded, it started throwing a lot of errors. Looking through the project, I remembered seeing some generic database access code in the project that we also had in many of our other projects, so I enabled it, tested the application, then sent it to QA.

QA tested it for a week and signed off on it. We deployed it to production on Monday. Tuesday morning, we had lots of errors in our logs. The stack traces were all the same. I go to the line indicated by the stack trace. This is the code I saw:
```
public static void closeDatabaseSession(Session session) {
    if (session == null){
        session.close();
    }
}
```
You read that correctly. Someone wrote code that throws an NPE 100% of the time and checked it in, and it has been in production (albeit unused) for several years. The reason QA and I didn't catch it is because the application works just fine until you hit your limit on DB sessions - and we didn't load test it to reveal this problem.

Moral of the story: Before you reactivate code that has been in production for years, make sure you read the code carefully for obvious bugs.

### One line = 200 hours of developer time
We bought a crap third-party product from a very large company whose name I shall not repeat here, then we hired a huge amount of crap contractors from India to make changes to it, with no in-house oversight or coding standards. It was the start of the project to move our entire policy system to a brand new, shiny platform where we could eliminate all the flaws and limitations of the current system, so everyone from appdev to infrastructure was hard at work just getting the stacks up and running. Needless to say, there was a lot of downtime in the first month as things were still being ironed out.

This was a third-party product with a separate frontend and backend, but the frontend depended on the backend even to be built and tested. I should also mention here that the contractors, for some reason, decided that nobody should be able to commit code to their single-branch repository except them. Yup, they had 40 contractors all working directly on master, and none of our in-house people were "allowed" to make changes.

One particular Friday, everything stopped working. My team of 3 developers and 2 analysts couldn't get into the frontend or test anything. This was normal for the early life of the project, as infrastructure issues were still being ironed out. We thought "maybe it will be up tomorrow". This went on for several days. The following Friday, after doing hardly any work for an entire week, I decided to find out what the problem could be.

I found the culprit within about 5 minutes. It was a single line of code in the backend that had a compile error. It had been there for the whole week. Keep in mind there is only one branch. Some g-tard wrote this code, saw that it wasn't compiling, and decided "Time to push this broken code to master". Then 40 contractors went on for 5 business days as if nothing was wrong.

This line of code was also completely useless - nobody needed it, so I commented it out. Tested it locally, everything worked again. So I pushed the change to the master branch. Then I wrote an email to the people running the project about how these "professional" contractors wasted 200 hours of my team's time because they had no coding standards and no understanding of how versioning software works (this was 2019, where if you can't use versioning, you shouldn't have a job in IT). This is after a few weeks of me complaining about them all committing unstable code to a master branch that our team depends on.

Needless to say, the contractors did not appreciate that email, and they tried to get me in trouble for committing to "their" repository without permission. One of the few good contractors on the team asked me how I even did it. The git rules on their repo require that you have a certain email address and use a specific commit message before it will allow you to push, and I had neither of those things.

To this day, none of them know that I have admin access on their repo and I disabled all the push rules for my commit.

### Internet Explorer sucks
As part of the same project, I got a defect assigned to me relating to a datepicker widget one one of the screens. This widget wasn't working in Internet Explorer (big surprise there). The user could type anything into it, but it wouldn't actually parse the date, so it would just send whatever the initial value was. To make matters more difficult, business wanted this datepicker to be inputmasked, so it would put in the appropriate special characters as you typed. The framework we were given didn't allow us to simply import one of the many freely available inputmask libraries out there because it was architected by a leperous chimpanzee, so I spent 3 days writing my own, while minding the limitations of Internet Explorer.

While doing this, I discovered several bugs in IE. Firstly, angular's $timeout didn't work in IE, at least not in this particular file. I never figured out why, so I made the code use window.setTimeout instead for IE users. Also, IE has a bug where, if an input is enabled, but it has a parent element that is disabled, the input will appear enabled and even let you type, but you cannot programatically set the carat's position (which is necessary for input masking). This was occurring because one of the engineers who built this wonderful framework misspelled something in one of the parent templates that wrapped this textfield, causing it to be permanently disabled. This wasn't a problem in Chrome or Firefox because those are real browsers who don't care if a span 3 levels up is disabled or not.

Anyway, I might have closed this defect earlier if IE were able to process redirects properly. This awesome framework's login process can't be drawn on a protocol diagram because it contains branching looper doopers and the redirect process is almost as complicated as human metabolism. It's so complicated that it was impossible to test the application locally using IE. To test IE changes, we had to write the code, commit it, build it, deploy it, then use IE to visit the test server. This is a 30 minute process, largely due to the fact that this product is a bloated piece of garbage. I made a small html file where I tested most of the code, but I still required small changes near the end that I had to put in the actual project. It took us 2 days to make 5 or 6 small fixes to close out this defect.

Moral of the story: Don't support IE. It's not a browser.

### Multithreading with statics
When I started at the company, one of my first projects was to experiment with WebDriver automation. I started playing with WebDriver through Java, and the guy sitting next to me was going the SeleniumIDE (javascript) route. We had a friendly competition each day to see who could do the most impressive thing with these tools.

Fast-forward a few years. He has circled around and is now working with a WebDriver-based testing tool written by a third-party. Our company hired a bunch of contractors for 2 months to write automation tests in this tool for one of our applications. My buddy (we will call him Chris) has been given the code for both, and periodically calls me over to look at some of it. Here is our story. BONG BONG!

Firstly, this tool (which is called Cognizant) is utter fucking garbage. No documentation came with it. You don't run tests through command line. You run tests only through the UI. One of the first things Chris asked the lead contractor was "How do I run these tests via command line?" This is necessary if you want to run it on some automated system like CRON or Jenkins. The answer he got was "Look at CommandLineArgs.java". This file was a huge mess of nonsense, with no documentation, and no usage comments. The parsing of command line args only started in this file, and spanned about 6 other files. We spent over an hour digging through the code to find the syntax for JUST ONE command.

After more persistence, he finally got one of the contractors to give him a script that would run the tests from command line. So, he set up his CI job to run the tests at a regular interval, and noticed that some of the tests would randomly fail, for no apparent reason. After a few days of fighting with this, he called me to take a look.

Chris explained to me that when he runs tests one-at-a-time, they always work. When he runs them in parallel, they randomly fail. Of course, the first thing that comes to mind is that something isn't being synchronized properly. We add some logging to the "fully-features" third-party product and narrow our search.
