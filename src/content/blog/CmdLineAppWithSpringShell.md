---  
title: "Command line tool using Spring Shell"  
description: "I built a command line tool using spring shell"  
date: "June 19 2024"  
---  
> Note: I am no expert, and all my posts are purely based on my experience.  
### Problem :
At work, we have a CICD pipeline that runs test cases for each PR and gives the result in the form of an HTML file compressed into a zip file. This zip file has HTML for every test suite that has run and its results. The worst part is that when a test case fails, we just get the output from the pipeline, like 30 test cases failed, but it doesn't provide any info on what all test cases have failed. So, I have to go through the zip file that has 100's of test suites, find index.html (the file that has the results), and find the test cases that have failed. Personally, I find writing and debugging code much easier than navigating through tonnes of folders to check for failures. So, I decided to write a simple parser that will parse, walk through all the folders, find the index.html (result file), extract the result, and display it in an easy and simple format like this...

    Total test cases failed: 8
    Test cases that has failed:
	    TestClassName
	    TestCaseName
	    
### Soultion :  
It's not always best to start out from scratch, but it depends on what we really want. In my case, I just need a simple programme that will do the above-mentioned job. So how can I get it done in the fastest way possible? It's using templates or frameworks. When I started out to write this app, I was arguing with myself about the choice of language. I need the app to be

 1. **Fast :**
I don't want to wait from 10 second for the app to startup and summaries the result for me.
2. **No Dependencies:**
Nobody wants to install all the dependencies just for this petty task. it must work just out of box.
3. **Less resource intensive:**
Same noone wants a test summariser to hold a 100MB or 200MB of memory.

On evaluating these categories, the first language that came to mind was GO. its light weight (Go routines), builds to a single-fat binary, and works out of the box. It's the perfect language for this use case.

but...

I chose java 🌚

**Why ?!**
Well, for a long time I wanted to learn spring and what's more good opportunity to learn spring than this? and best way to learn is to get the dirt on your hands. So, I googled what's the best framework or lib for writing a command line tool. Spring Shell was most recommended, so I used this [spring initializer](https://start.spring.io/) with below option to get a spring template with spring shell dependency.
![Spring init](https://i.imgur.com/JB5WU8e.png)

Spring Shell is a command line framework, like every other framework, it provides all the basic things that we might need for making a cmd line app like TUI (Terminal UI). it helped me bootstrap the app, now all I have to do is to add the commands and their logic to it.

    ├───src
    │   ├───main
    │   │   ├───java
    │   │   │   └───com
    │   │   │       └───esb
    │   │   │           └───FIleWalker
    │   │   │                   FIleWalkerApplication.java
    │   │   │                   ShellCommand.java
    │   │   │                   ZipFileProcessor.java
This is the file structure that came with the initializer except the ShellCommand.java, ZipFileProcessor.java
**ShellCommand.java** is where I define the commands that I am about to use. **ZipFileProcessor.java** is where I put the logic for the commands.
let me start with the ShellCommand.java

    @ShellComponent  
    public class ShellCommand {  
      
        @Autowired  
      ZipFileProcessor zipProcessor;  
      
        @ShellMethod("reads and gather info")  
        public void processZipFile(@ShellOption String filePath,@ShellOption String htmlId){  
            try{  
                zipProcessor.process(filePath,htmlId);  
            } catch (IOException e) {  
                System.err.println(e.getLocalizedMessage());  
                return;  
            }  
            System.out.println("Total Test cases failed: " + String.valueOf(zipProcessor.totalFailCount));  
            for(Map.Entry<String, Long> entry : zipProcessor.failedTest.entrySet()){  
                if(entry.getValue() > 0) {  
                    System.out.println(entry.getKey() +" ---> "+ entry.getValue().toString());  
                }  
            }  
            System.out.println("Testcases that are failed: ");  
            zipProcessor.hrefs.stream().forEach(entry -> {  
                System.out.println(entry.toString());  
            });  
        }  
    }
if you look closely there are few things that are few new things, while rest is just a plain java code.
Here is list of things that are new in the above code,

 - @Autowired :
 This annotation is used for dependency injection in Spring. It allows Spring to automatically inject the required dependencies into a class. 
 
Meaning, ZipFileProcessor class gets injected here with all it's methods and variables. So, if I need the same somewhere else, we can right away we could inject it directly the required methods. And I got a question here, how's it advantages than importing the zipProcessor ? I guess, spring considers the zip processor as singleton object and by injecting, the same object is injected everywhere, and the object is shared by multiple methods.

 - @ShellComponent 
This annotation is used to mark a class as a Spring Shell component. It allows the class to define command-line commands that can be executed in the Spring Shell environment. Classes annotated with `@ShellComponent` can define methods annotated with `@ShellMethod` to create shell commands.

 - @ShellMethod 
This annotation is used to mark a method as a command that can be executed in the Spring Shell environment. The annotation value is a description of the command. The method can have parameters annotated with `@ShellOption` to define command-line options.

 - @ShellOption
This annotation is used to define options (parameters) for a Spring Shell command. It specifies that the annotated method parameter is a command-line option.

Now that we have defined those commands, we can move the logic part. ZipFileProcessor.java simply navigates through the zipfile without decompressing and parse the index.html using **Jsoup**. Once we get the needed data from the html, we store in the class variable which will be injected too.

    @Service  
    public class ZipFileProcessor {  
      
        public long totalFailCount = 0;  
        public Hashtable<String,Long> failedTest = new Hashtable<>();  
        List<String> hrefs = new ArrayList<>();  
      
        public void process(String filePath, String htmlClass) throws IOException {  
            ZipFile zf = new ZipFile(filePath);  
            zf.stream().filter(zipEntry -> zipEntry.getName().endsWith("index.html")).forEach(entry -> {  
                try {  
                    InputStream is = zf.getInputStream(entry);  
                    long fails = extractInfo(is,htmlClass);  
                    failedTest.put(entry.getName(),fails);  
                    totalFailCount += fails;  
                } catch (IOException e) {  
                    throw new RuntimeException(e);  
                }  
            });  
        }  
      
        public long extractInfo(InputStream is,String htmlClass) throws IOException {  
            Document doc = Jsoup.parse(is,null,"");  
            Element failCounter = doc.getElementById(htmlClass);  
            Elements divs = doc.select("div#tab0");  
            for (Element div : divs) {  
                Elements h2s = div.select("h2:contains(Failed tests)");  
                if(!h2s.isEmpty()){  
                    Elements links = div.select("li a");  
                    for(Element link : links){  
                        hrefs.add(link.attr("href"));  
                    }  
                }  
            }  
            if(failCounter != null) {  
                String count = failCounter.getElementsByClass("counter").first().text();  
                return Long.parseLong(count);  
            }  
            throw new IOException("No InfoBox found for failures");  
        }  
    }
- @Service
The `@Service` annotation marks the class as a Spring component, so it gets picked up by component scanning. This means that Spring will automatically detect and register this class as a bean in the application context. It can be injected into other Spring components using dependency injection. For example, a service class can be injected into a Spring Shell command component to provide business logic.

And the app is ready to serve the need, we just give the file path and html class under which the required info is present, and the spring shell will fetch it for us.
Here is the output of the app,

      .   ____          _            __ _ _
     /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
    ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
     \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
      '  |____| .__|_| |_|_| |_\__, | / / / /
     =========|_|==============|___/=/_/_/_/
    
     :: Spring Boot ::                (v3.3.0)
    
    2024-06-19T15:15:55.524+05:30  INFO 17744 --- [FIleWalker] [           main] c.esb.FIleWalker.FIleWalkerApplication   : Starting FIleWalkerApplication v0.0.1-SNAPSHOT using Java 17.0.11 with PID 17744 (C:\Users\ranku\side\FIleWalker\target\FIleWalker-0.0.1-SNAPSHOT.jar started by ranku in C:\Users\ranku\side\FIleWalker\target)
    2024-06-19T15:15:55.532+05:30  INFO 17744 --- [FIleWalker] [           main] c.esb.FIleWalker.FIleWalkerApplication   : No active profile set, falling back to 1 default profile: "default"
    2024-06-19T15:15:56.633+05:30  WARN 17744 --- [FIleWalker] [           main] org.jline                                : Unable to create a system terminal, creating a dumb terminal (enable debug logging for more information)
    2024-06-19T15:15:57.034+05:30  INFO 17744 --- [FIleWalker] [           main] c.esb.FIleWalker.FIleWalkerApplication   : Started FIleWalkerApplication in 2.186 seconds (process running for 2.936)
    Total Test cases failed: 8
    tests/server/staging/reports/tests/integrationTest/index.html ---> 8
    Testcases that are failed: 
    classes/com.example.is.KarateRunner.html
    classes/com.example.is.KarateRunner.html#testParallel
    classes/com.exm.net.socket.SocketPoolTest.html
    classes/com.exm.net.socket.SocketPoolTest.html#testHTTPS_Direct_NoSocketPool
    classes/com.exm.net.socket.SocketPoolTest.html
    classes/com.exm.net.socket.SocketPoolTest.html#testHTTPS_Direct_SocketPool
    classes/client.HTTPTest.html
    classes/client.HTTPTest.html#testHTTPClientURLWithHostNameVerification
    classes/client.HTTPTest.html
    classes/client.HTTPTest.html#testPIE82363
    classes/exm.server.PIE_70271Test.html
    classes/exm.server.PIE_70271Test.html#testOnErrorMissingMessageService
    classes/exm.server.PIE_70271Test.html
    classes/exm.server.PIE_70271Test.html#testOnErrorMissingMessageStatus
    classes/exm.server.PIE_70271Test.html
    classes/exm.server.PIE_70271Test.html#testOnErrorSuccess


### Pain :
Working with Spring was smooth, and most things worked out of the box, as Java has almost every method and library for almost everything, but setting up the environment was the most dreadful part of the development. I have built the app into a jar for Java 17 to run.  

The worst part is that the app takes 8 seconds to complete the fetch, which is quite bad. but it's reasonable as the JVM has to startup -> spring boot starts -> spring shell -> program runs. i believe that spring boot startup and JVM start-up take up most of the time.

### Work Around :   Avoid using JAVA for petty things ?
### Conclusion :  
Spring Shell is pretty solid framework, has good community and Stack overflow post for literally everything. If you are writing enterprise software, it makes sense to write it using spring. But for small use case like these, languages like Lua or go would be great choice. Hope you had nice read and feel free add your thoughts in the comments, see you on next post. Thanks!
 
Bye 👋

