# CrappyCode
This is a collection of programming horror stories that I have encountered over the years. Most of these were found as I was working at an insurance company.

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
Yup. You read that correctly. Someone wrote code that throws an NPE 100% of the time and checked it in. The worst part is, we have this same file in many other projects, and they don't have this bug. The reason me and QA didn't catch it is because the application works just fine until you hit your limit on DB sessions - and we didn't load test it to reveal this problem.

Moral of the story: Before you use code that has been in production for years but unused, make sure you read the code carefully for obvious bugs.

### dsf
