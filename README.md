# CrappyCode
This is a collection of programming horror stories that I have encountered over the years. Most of these were found as I was working at an insurance company. They are face-palmy enough that I decided to share them. Perhaps you can learn some lessons from these stories, just as I have.

### The Case of the Uncommitted, Untested, Unoptimized Code
We owned a batch job that ran daily. Until this incident, we'd never had a problem with it. One day, it started throwing OutOfMemory Exceptions, and it threw the same exceptions the next day. I looked at the stack trace to find the line it was blowing up on (as you do), and I opened up my Subversion client to check out the code.

When I opened the file to see where the exception was being thrown, I immediately knew this was going to take longer than expected, because the line throwing the exception was blank - it was whitespace. This means one and only one thing - the code from SVN was not the code that was in production. Apparently, the dingus who wrote this job didn't believe in version control or deployment pipelines, and the last time it was built, he built it from his local machine but never checked the code back in.

So began a decompiling spree. Luckily this batch job was relatively simple, so I only had about 15 java files to recreate and fix. Once I rebuilt the project from production bytecode, I tried finding out how he tested it. I asked some of my teammates if they knew how it was tested, and apparently the snowflake who wrote it didn't believe in having a test environment. He tested the thing in production. So, I had to add some config features to allow switching between environments.

Once I finally got the thing running against our test system, I learned that this thing was pulling hundreds of thousands of database rows into memory during its initial query. This job is supposed to grab data from one database, do some transformations, and drop it into another database. The first step is where the OutOfMemory was happening. I ended up talking to someone who better understood the tables this batch job was reading from. Together, we learned that this query was doing not one, not two, not three, but FOUR redundant table joins. This means the result set it was extracting was 4 orders of magnitude larger than it needed to be, entirely filled with duplicate rows. Apparently the translucent jellyfish who wrote this thing didn't understand how relational databases work or how to write sql queries. This was also the reason the batch job (which runs every day) was taking 18 hours to finish, until it stopped working.

Once we fixed the query, we ran the job. It finished in 30 seconds.

### Beware of version upgrades
We had an initiative at the company to keep as many similar applications on the same tech stack as possible (eg all webservices would use the same container and framework). We were migrating a service from one container to another, and in the process we had to update some versions of the database access library. Some of the code in this particular project only worked on the older version - when we upgraded, it started throwing a lot of errors. Looking through the project, I remembered seeing some generic database access code in the project that we also had in many of our other projects, so I enabled it, tested the application, then sent it to QA.

QA tested it for a week and signed off on it. We deployed it to production on Monday. Tuesday morning, we had lots of errors in our logs. The stack traces were all the same. I go to the line indicated by the stack trace. This is the code I saw:
```
public static void closeDatabaseSession(Session session) {
    if (session == null){
        session.close();
    }
}
```
You read that correctly. Someone wrote code that throws an NPE 100% of the time and checked it in, and it has been in production (albeit unused) for several years. The reason QA and I didn't catch it is because the application works just fine until you hit your limit on DB sessions - and we didn't load test it to reveal this problem.

Moral of the story: Before you reactivate code that has been in production for years, make sure you read the code carefully for obvious bugs. Or, maybe just don't check in broken code in the first place.

### One line = 200 hours of developer time
The company started a massive project to move our entire policy system to a brand new, shiny platform where we could eliminate all the flaws and limitations of the current system. Upper management, in their infinite wisdom, decided that the best way to accomplish this was to buy a crap third-party system off the shelf, and hire bulk contractors from India to modify it, with no in-house oversight or coding standards. After the contractors leave, upper management planned to dump the code onto a bunch of maintenance people who aren't on the project and had no training with the product, because "That's how big companies operate."

This third-party product had a separate frontend and backend, but the frontend depended on the backend even to be built and tested. I should also mention here that the contractors, for some reason, decided that nobody should be able to commit code to their single-branch repository except them. Yup, they had 40 incompetent contractors all working directly on master, and none of our in-house people were "allowed" to make changes. But "that's how big companies operate."

One particular Friday, everything stopped working. My team of 3 developers and 2 analysts couldn't get into the frontend or test anything. This was normal for the early life of the project, as infrastructure issues were still being ironed out. We thought "maybe it will be up tomorrow". This went on for several days. The following Friday, after doing hardly any work for an entire week, I decided to find out what the problem could be.

I found the culprit within about 5 minutes. It was a single line of code in the backend that had a compile error. It had been there for the whole week. Keep in mind there is only one branch. Some sockpuppet contractor wrote this code, saw that it wasn't compiling, and decided "Time to push this broken code to master". Then 40 contractors went on for 5 business days as if nothing was wrong.

This line of code was also completely useless - nobody needed it, so I commented it out. Tested it locally, everything worked again. So I pushed the change to the master branch. Then I wrote an email to the people running the project about how these "professional" contractors wasted 200 hours of my team's time because they had no coding standards and no understanding of how versioning software works (this was 2019, where if you can't use versioning, you shouldn't have a job in IT). This is after a few weeks of me complaining about them constantly committing unstable code to a master branch that our team depends on.

Needless to say, the contractors did not appreciate that email, and they tried to get me in trouble for committing to "their" repository without permission. One of the contractors on the team asked me how I even did it. The git rules on their repo require that you have a certain email address and use a specific commit message before it will allow you to push, and I had neither of those things.

To this day, none of them know that I have admin access on their repo and I disabled all the push rules for my commit.

### Internet Explorer sucks
As part of the same project, I got a defect assigned to me relating to a datepicker widget one one of the screens. This widget wasn't working in Internet Explorer (big surprise there). The user could type anything into it, but it wouldn't actually parse the date, so it would always store whatever the initial value was, regardless of what you typed. To make matters more difficult, business wanted this datepicker to be inputmasked, so it would put in the appropriate special characters as you typed. The framework we were given didn't allow us to simply import one of the many freely available inputmask libraries out there because it was architected by a leperous chimpanzee, so I spent 3 days writing my own, while minding the limitations of Internet Explorer.

While doing this, I discovered several bugs in IE. Firstly, angular's $timeout didn't work in IE, at least not in this particular file. I never figured out why, so I made the code use window.setTimeout instead for IE users. Also, IE has a bug where, if an input is enabled, but it has a parent element that is disabled, the input will appear enabled and even let you type, but you cannot programatically set the carat's position (which is necessary for input masking). This was occurring because one of the engineers who built this wonderful framework misspelled something in one of the parent divs that wrapped this textfield, causing it to be permanently disabled.

Anyway, I might have closed this defect earlier if IE were able to process redirects properly. This awesome framework's login process can't be drawn on a protocol diagram because it contains branching looper doopers and the redirect process is almost as complicated as human metabolism. It's so complicated that it was impossible to test the application locally using IE. To test IE changes, we had to write the code, commit it, build it, deploy it, then use IE to visit the test server. This is a 30 minute process, largely due to the fact that this product is a bloated piece of garbage. I made a small html file where I tested most of the code, but I still required small changes near the end that I had to put in the actual project. It took us 2 days to make 5 or 6 small fixes to close out this defect.

Moral of the story: Don't support IE. It's not a browser. If business tells you to support IE, don't. When they say "but a lot of our users use IE," respond with "Microsoft told them not to."

### Multithreading with statics
A co-worker I used to sit next to was assigned to work on a WebDriver-based testing tool written by a third-party. Our company hired a bunch of contractors for 2 months to write automation tests in this tool for one of our applications. My buddy (we will call him Chris) has been given the code for both, and periodically calls me over to look at some of it. Here is our story. BONG BONG!

Firstly, this tool (which is called Cognizant Intelligent Test Scripter) is utter fucking garbage. No documentation came with it. You don't run tests through command line. You run tests only through the UI. One of the first things Chris asked the lead contractor was "How do I run these tests via command line?" This is necessary if you want to run it on some automated system like CRON or Jenkins. The answer he got was "Look at [Lookup.java](https://github.com/CognizantQAHub/Cognizant-Intelligent-Test-Scripter/blob/master/Engine/src/main/java/com/cognizant/cognizantits/engine/cli/LookUp.java)". This file was a huge mess of nonsense, with no documentation, and no usage comments. The parsing of command line args only started in this file, and spanned about 6 other files. We spent over an hour digging through the code to find the syntax for JUST ONE command.

After more persistence, he finally got one of the contractors to give him a script that would run the tests from command line. So, he set up his CI job to run the tests at a regular interval, and noticed that some of the tests would randomly fail, for no apparent reason. After a few days of fighting with this, he called me to take a look.

Chris explained to me that when he runs tests one-at-a-time, they always work. When he runs them in parallel, they randomly fail. Of course, the first thing that comes to mind is that something isn't being synchronized properly. We add some logging to the "fully-featured" third-party product and narrow our search. I laughed out loud when I discovered the problem.

When you do WebDriver automation, your code keeps a WebDriver object which acts as a handle to the browser. This is how you send commands like "type something in this text field" or "go to this URL". This fully-featured product, which is advertized as being able to run tests in parallel, contained only one WebDriver reference, and [it was static](https://github.com/CognizantQAHub/Cognizant-Intelligent-Test-Scripter/blob/master/Engine/src/main/java/com/cognizant/cognizantits/engine/core/Control.java#L53). This means if you run one test at a time, everything works fine. If you run, say, 5 tests at once, they will each replace the reference to that handle, and then the first test to complete will close the browser window of the last test that started, causing it to fail.

The repo in those links has a lot of bad code in it. It's a great way to study how NOT to write a program.

The tests written by the contractors were even worse. We found such gems as (comments mine)
```
for (int i=1 ; i<=2 ; i++){
    if (i==1){
        //do something
    } else if (i==2){
        //do something else
    }
}
```
and
```
for (int i=1 ; i<=3 ; i++){
    switch (i){
        case 1: doSomething(i-1);break;
        case 2: doSomething(i-1);break;
        case 3: doSomething(i-1);break;
    }
}
```
and (comments mine)
```
public boolean validate(int input){
    String[] values = {"0","1","2"}; //length=3
    if (input <= values.length){
        setSomething(values[input]); //throws exception if input==3
        return true;
    } else {
        throw new ArrayIndexOutOfBoundsException(); //make the above if statement irrelevant
    }
}
```

### An interview that went off the rails
We had a guy (we will call him Dave) apply for a Senior Engineer position on our team. His resume looked pretty good, 9+ years of java, plentiful frontend experience. We did a phone screen and he answered all of the questions satisfactorily, so we brought him in for an in-person interview. COVID-19 had just struck, so we ended up doing a call-in interview with one of those shared text boxes for writing code. Remember: Senior engineer position, 9+ years of Java experience.

I asked the first question, which I thought was too easy (considering the position he applied for): Write a java method that determines if an integer is prime. I would expect someone with 9+ years of java experience to be able to bang this out in 3-5 minutes, including the time they take to explain their process.

20 minutes later, Dave finished coding it. It was wrong. It also wasn't optimized (that's usually the second half of the exercise). 9+ years of Java experience. My ass.

Dave spent the first 5 minutes of the exercise googling what a prime number is. 9+ years Java experience. Technical background. Didn't understand prime numbers.

Next, my manager asked some design questions: Design a database and a service to manage user logins. This is a pretty standard thing: Create your user table, a security questions table, and a login history table, and an API to get data from it. It took Dave so long to design the tables that we never got to the service API part of the question. His tables weren't even normalized.

He wanted to store the password in the user table, so I asked how. He said Spring would do it (fair enough), so I asked him how Spring decided whether your password is correct when you login (i wanted him to explain a little bit about encryption and/or salting). He bullshitted the answer, and was wrong of course. His resume included Spring Security as a list of things he knew very well. Bullshit.

My manager asked Dave to write a query that would list every user that logged in today. This was a hilarious question, because Dave designed his tables in a way that made this query impossible to produce. The first thing he did of course, is google "sql join". Then he proceeded to write a query with a hard-coded cutoff date. When I asked him how comfortable he was with writing SQL in the phone screen, he said 7 out of 10. Bullshit.

The "work experience" section of his resume contained a company that both my manager and I used to work at. Our employment even overlapped with Dave's, so we asked him about it. We asked him what team he was on, to which he replied "I don't remember." That's weird, because his resume said he worked there for five years. We asked him what applications he worked with. All he could say was "Internal applications" and never actually named any. We asked him who his manager was for those 5 years. He said "Uhhh.... Stephanie." There was no Stephanee at that company. We even asked him the location of the office he supposedly drove to every weekday for 5 years, and he couldn't answer that either. My manager is a pretty mellow dude, and this is the first time I've seen him get really angry. Dave flat out made up part (or all) of his work experience, then wasted our time with how shit he was.

It wasn't a complete waste of our time, though. We were having too much fun in chat behind his back. A good hour or so of entertainment on behalf of this 9+ year senior engineer applicant who didn't know what prime numbers or table joins were. Our architect, who usually sits out of interviews, was present for this one and said
- Your guys are brutal... I'm going to use them for all my interviews :)
- seriously, thank for you convincing me to join
- if only I could get my glass of bourbon...it'd be perfect

### An interview in the era of ChatGPT
From all the interviews I've run, I've learned that people lying about their qualifications is sadly a regular thing. I've had people do the interview on their phone, and after each question, it's obvious that they're typing because of the very jarring phone wobble. We interviewed one guy who had large glasses, and we could identify which websites he was looking at by the reflection of his computer screen. But, there was one interviewee that stood out to me as the weirdest one...

To protect his identity we will call him Johnny (as you'll soon see, he was shit, so I don't remember his real name anyway). We get Johnny into a remote interview via webcam. My manager likes having me in interviews because I come up with questions that are almost impossible to google in less than 10 seconds - things that you would know instantly if you had experience on a particular topic but otherwise are kind of obscure.

Anyways, I start asking questions. Every time he gives an answer, it starts like this:
```
Ok, so, (repeat question), let's see, give me one moment to recollect, so, with regard to (repeat question)...
```
Every. Single. Answer. Chess.com detects cheaters by, among other things, looking at the timings players take to make moves. When the timings are pretty much identical regardless of position, the player is most likely cheating. Players who understand the game will take almost no time in certain situations when the gamestate is easy and the best move is obvious, but cheaters don't - they watch an AI play the same position and copy the move, which takes a few extra seconds. Likewise, in complicated positions, real players take much longer because they have to think. Cheaters don't know the difference between a simple position and a tough position, they just look at what the AI did on their other screen and copy it the same way they would with the easy moves.

The way Johnny answered these interview questions was exactly like the way cheaters play on chess.com. He ALWAYS took exactly 10 seconds after each question to vamp before actually attempting to say anything substantial, regardless of the difficulty of the question. When he did start speaking, it was just a flow of random buzz words that didn't make any sense.

The weird thing about this interview is that his hands were visible the whole time. He wasn't looking up any of the answers himself. My co-workers in the call thought he had a friend behind the camera looking things up for him, but I have a different theory based on something that happened. I asked some question (I wish I could remember the wording I used), and Johnny paused, leaned towards his screen with a confused look as he read something, and asked "Did you say (some word unrelated to programming)?" I think he had a voice to text program listening to us that was typing everything we said, and it misheard me saying a similar word, and it was sending that to ChatGPT or whatever tool he had. I turned the rest of the interview into a personal game where I tried to get his thing to make mistakes, so at least I had fun while my time was being wasted.

The depressing thing about stories like Johnny's is this: Their bullshit sometimes works. If it didn't, we wouldn't get so many candidates who lie about their experience or google the answers live. There must be so many companies out there whose interviewers know nothing about the job position, and are only listening for buzz words. The lesson to learn here is this: Put people in your interviews who actually do the work. Don't send an emissary of ignorance who can be dazzled by words on their checklist. You will end up hiring the equivalent of a toddler - someone who is incapable of doing any work, but is still really expensive to have around.
