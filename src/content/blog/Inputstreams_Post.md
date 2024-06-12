---  
title: "Java InputStreams and How I Have Used It."  
description: "How I have used Inpustream in my work"  
date: "June 12 2024"  
---  
> Note: I am no expert, and all my posts are purely based on my experience.  
### Problem :

Recently, I was working on mailing services module called MIME, and there's a secure version of it called SMIME. Both of these services are used to automate or define mailing tasks that should happen on your web server. For example. Consider the below flow of events.

 !['o' output](https://i.imgur.com/AjrnbNm.png)
 
This is a simple mail service that is commonly used to send the bill to customers. And these can be built in no code way; There are in-built services that can easily build this flow for you.

These services were good, but they were all implemented in such a way that they always loaded up the **entire body content into memory**. which is a good fast way to get things done when the files are smaller, but when the files get bigger, like 50 MB or above, there is a good chance of running into a Java heap space error, making the service flaky (works sometimes but fails sometimes). So, we have to make sure that the entire data isn't loaded into memory. This enhancement might seem simple, but in a codebase, this module hasn't been touched for like 30 years, making it much more difficult.  

### Soultion :  

>Lets call **temporary space** as Tspace  
  
Well, the solution is simple. We write a copy of the file that we are trying to load into mail into a **temporary space** and then get a Inputstream of the file in the **Tspace**. This resolved the issue, but the code needed 250 lines for refactoring, which nearly broke all the existing test cases. I had to learn each scenarios one by one from the test cases and refactor the code accordingly.

Interesting take on this, whenever we are dealing with heavy files, inputstream comes in handy. There are different types of input streams and different varieties of input streams. Each one serves its own niche cases. There are a few quite popular ones, like ByteArrayInputstream, which is nothing but creating an input stream with a byte array as the source. So technically, you open an inputstream to a byte array that is loaded in memory. I feel InputStream is like a portal to a storage room; you put all your stuff there and get it bit by bit whenever you need it.

### Pain  
As much as these can be useful, they must be used with care, and we shouldn't do a few things with them.  
Let me give an example:

    public static InputStream getStreamFromFile(String filePath) throws Exception {  
	    InputStream data = null;  
	    try {  
		    // Read in the streams  
		    data = new BufferedInputStream(  
		    new FileInputStream(filePath), BUFFER_SIZE);  
	    } catch (IOException e) {  
		    Server.logError(e);  
	    }  
		    return data;  
    } 

 
In the above example, there is a serious bug. This method gets an input stream for the mentioned file and passes it to some other function that does something with it. This is a really bad design. Why? Well, the file for which you created an input stream undergoes locking. So, this file is now locked and can't be accessed by other processes until you close the input stream. To make it worse, we pass it across to the function that will be using it. so you don't really know where to close. Closing the input stream, which is being used across many functions, has many risks; some methods might be using it and some might not, so closing it will cause errors across all those methods.

### Work Around :  
I faced the above problem in one of my customer issues, where the customer was using the file pooling mechanism to move files to a separate folder once they were processed. The integration server was using a similar method in a service to get the input stream for that file. This input stream is passed across many methods. So, closing the input stream at the last method it is passed to is not a good idea, as it might be used somewhere else. So, I tried to use this ***[AutoClosableInputStream](https://commons.apache.org/proper/commons-io/apidocs/org/apache/commons/io/input/AutoCloseInputStream.html), but it wasn't the right fit as the input stream is only closed when the input stream is closed completely*** and in my case, the input stream is given to the user. So, we don't know when the user will read it fully. Using "Finally" to close it is also out of choice, as we never know when the input stream will be closed.

!['o' output](https://i.imgur.com/MGDYv8E.png)

And the only good option is to use try-with-resource. Here, we try to open the resource that we want and it is opened successfully, then we execute the try block, and once it is used, like losing the last ref, the input stream is automatically closed. Check out the below code.

    try (FileInputStream fis = new FileInputStream(filePath)) {  
	    // read the file into byte array to prevent file locking.  
	    fileBytes = Streams.readFully(fis);  
	    data = new ByteArrayInputStream(fileBytes);  
    } catch (IOException e) {  
	    Server.logError(e);  
    }  
    return data; 

 
In the above code, we try with the resource, and this will take care of closing the stream. Also, note that we are loading the entire file into memory by reading the input stream fully and creating a ByteArrayInputStream out of the byte array. With this byte array as the source, the input stream can be passed around and processed without locking the file, as we have a copy in the heap memory. But if the file size is larger than the available heap space, the service would fail with a Java heap space error.

### Conclusion :  
InputStreams are a good tool to have in your tool kit. Some important things to keep in mind are:

- Inpustream can lead to the locking of resources.  
- It should be used and closed in the same way as when you are opening it.  
- It can be space efficient as we will be processing data byte by byte.

Anyways, I think you might have a basic idea of input streams and how I have used them. If you think there could be a better way to use them or if I have missed something interesting about input streams, please let me know in the comments. Thanks! Have a nice day.