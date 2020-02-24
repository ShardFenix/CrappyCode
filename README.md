# CrappyCode
This is a collection of programming horror stories that I have encountered over the years. Most of these were found as I was working at an insurance company. They are face-palmy enough that I decided to share them. Perhaps you can learn some lessons from these stories, just as I have.

### The Case of the Uncommitted, Untested, Unoptimized Code
We owned a batch job that ran daily. Until this incident, we'd never had a problem with it. One day, it started throwing OutOfMemory Exceptions - likely due to our rapidly growing customer base. I looked at the stack trace to find the line it was blowing up on (as you do), and I opened up my Subversion client to check out the code.

When I opened the file to see where the exception was being thrown, I immediately knew this was going to take longer than expected, because the line throwing the exception was blank - it was whitespace. This means one and only one thing - the code from SVN was not the code that was in production. Apparently, the person who wrote this job didn't believe in version control or deployment pipelines, and the last time it was built, he built it from his local machine but never checked the code back in.

So began a decompiling spree. Luckily this batch job was relatively simple, so I only had about 15 java files to recreate and fix. Once I rebuilt the project from production bytecode, I tried finding out how he tested it. I asked some of my teammates if they knew how it was tested, and apparently the person who wrote it didn't believe in having a test environment. He tested the thing in production. So, I had to add some config features to allow switching between environments.

Once I finally got the thing running against our test system, I learned that this thing was pulling hundreds of thousands of database rows into memory during its initial query. This job is supposed to grab data from one database, do some transformations, and drop it into another database. The first step is where the OOM was happening. I ended up talking to someone who better understood the tables this batch job was reading from. Together, we learned that this query was doing not one, not two, not three, but FOUR redundant table joins. This means the result set it was extracting was 4 orders of magnitude larger than it needed to be, entirely filled with duplicate rows. Apparently the person who wrote this thing didn't understand how relational databases work or how to write sql queries. This was also the reason the batch job (which runs every day) was taking 18 hours to finish, until it stopped working.

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
Yup. You read that correctly. Someone wrote code that throws an NPE 100% of the time and checked it in. The reason QA and I didn't catch it is because the application works just fine until you hit your limit on DB sessions - and we didn't load test it to reveal this problem.

Moral of the story: Before you reactivate code that has been in production for years, make sure you read the code carefully for obvious bugs.

### One line = 200 hours of developer time
We bought a crap product from a very large company whose name I shall not repeat here, then we hired a huge amount of crap contractors to make changes to it, with no in-house oversight or coding standards. It was the start of the project to move our entire policy system to a shiny new turd, so everyone from appdev to infrastructure was hard at work just getting the stacks up and running. Needless to say, there was a lot of downtime, which is to be expected.

This was a third-party product with a separate frontend and backend, but the frontend depended on the backend to work. The way it was designed, we needed the backend running just to build the frontend dist. I should also mention here that the contractors, for some reason, decided that nobody should be able to commit code to their single-branch repository except them. Yup, they had 40 contractors all working directly on master, and nobody in-house was "allowed" to make changes.

One particular Friday, everything stopped working. My team of 3 developers and 2 analysts couldn't get into the app or test anything. This was normal for the early life of the project, we thought "maybe it will be up tomorrow". This went on for several days. The following Friday, after doing hardly any work for an entire week, I decided to find out what the problem could be.

I found the culprit within about 5 minutes. It was a single line of code that was guaranteed to throw a null pointer exception. This line of code was also completely useless - nobody needed it, so I commented it out. Tested it locally, everything worked again. So I pushed the change to the master branch. Then I wrote an email to the people running the project about how these contractors wasted 200 hours of my team's time because they had no coding standards and no understanding of how versioning software works (this was 2019, where versioning software is used in 100% of IT shops), and that it took me under 10 minutes to resolv the issue with no side-effects.

Needless to say, the contractors did not appreciate that email, and they tried to get me in trouble for committing to "their" repository without permission. One of the good contractors who I actually like asked me how I even did it. The git rules on their repo require that you have a certain email address and use a specific commit message before it will allow you to push, and I had neither of those things.

To this day, none of them know that I have admin access on their repo and I disabled all the rules for my commit.
