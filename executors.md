## Runnable

### task

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

public class Server {
    private ThreadPoolExecutor executor;

    public Server() {
        executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
    }

    public void executeTask(Task task) {
        System.out.printf("Server: A new task has arrived\n");
        executor.execute(task);
        System.out.printf("Server: Pool Size: %d\n", executor.getCorePoolSize());
        System.out.printf("Server: Active Count: %d\n", executor.getActiveCount());
        System.out.printf("Server: completed Tasks: %d\n", executor.getCompletedTaskCount());
    }

    public void endServer() {
        executor.shutdown();
    }
}
```

### server

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

public class Server {
    private ThreadPoolExecutor executor;

    public Server() {
        executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
    }

    public void executeTask(Task task) {
        System.out.printf("Server: A new task has arrived\n");
        executor.execute(task);
        System.out.printf("Server: Pool Size: %d\n", executor.getCorePoolSize());
        System.out.printf("Server: Active Count: %d\n", executor.getActiveCount());
        System.out.printf("Server: completed Tasks: %d\n", executor.getCompletedTaskCount());
    }

    public void endServer() {
        executor.shutdown();
    }
}
```



### main

```java
public class Main {
    public static void main(String[] args) {
        Server server = new Server();
        for(int i = 0; i < 100; i++) {
            Task task = new Task("Task" + i);
            server.executeTask(task);
        }
        server.endServer();
    }
}

```

## Callable

### FactorialCalculator

```java
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class FactorialCalculator implements Callable<Integer>{
    private Integer number;
    public FactorialCalculator(Integer number) {
        this.number = number;
    }

    @Override
    public Integer call() throws Exception {
        int result = -1;
        if((number == 0) || (number == 1)) {
            result = 1;
        } else {
            for(int i = 2; i<=number; i++) {
                result *= i;
                TimeUnit.MILLISECONDS.sleep(20);
            }
        }

        System.out.printf("%s: %d\n", Thread.currentThread().getName(), result);
        return result;
    }
}
```

### Main

```java
import java.util.ArrayList;
import java.util.Random;
import java.util.concurrent.*;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(2);

        List<Future<Integer>> resultList = new ArrayList<>();

        Random random = new Random();
        for(int i=0; i<10; i++) {
            Integer number = random.nextInt(10);
            FactorialCalculator calculator = new FactorialCalculator(number);
            Future<Integer> result = executor.submit(calculator);
            resultList.add(result);
        }

        do {
            System.out.printf("Main: Number of Completed Tasks: %d\n", executor.getCompletedTaskCount());
            for(int i=0; i<resultList.size(); i++) {
                Future<Integer> result = resultList.get(i);
                System.out.printf("Main: Task %d: %s\n", i, result.isDone());
            }

            try {
                TimeUnit.MILLISECONDS.sleep(50);
            } catch(InterruptedException e) {
                e.printStackTrace();
            }
        } while(executor.getCompletedTaskCount() < resultList.size());

        System.out.printf("Main: Results\n");
        for(int i = 0; i < resultList.size(); i++) {
            Future<Integer> result = resultList.get(i);
            Integer number = null;
            try {
                number = result.get();
            } catch(InterruptedException e) {
                e.printStackTrace();
            } catch(ExecutionException e) {
                e.printStackTrace();
            }
            System.out.printf("Main: Task： %d: %d\n", i, number);
        }
        executor.shutdown();
    }
}
```

## Running multiple tasks and process the first result

### TaskValidator

```java
import java.util.concurrent.Callable;

public class TaskValidator implements Callable<String> {
    private String user;
    private String password;
    private UserValidator validator;

    public TaskValidator(UserValidator validator, String user, String password) {
        this.validator = validator;
        this.user = user;
        this.password = password;
    }

    @Override
    public String call() throws Exception {
        if(!validator.validate(user,password)) {
            System.out.printf("%s: The user has been found\n",validator.getName());
        }
        return validator.getName();
    }
}
```

### UserValidator

```java
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class UserValidator {
    private String name;

    public UserValidator(String name) {
        this.name = name;
    }

    public boolean validate(String name, String password) {
        Random random = new Random();
        try {
            long duration = (long)(Math.random()*10);
            System.out.printf("Validator %s: Validating a user during %d seconds\n", this.name, duration);
            TimeUnit.SECONDS.sleep(duration);
        } catch(InterruptedException e) {
            return false;
        }

        return random.nextBoolean();
    }

    public String getName() {
        return name;
    }
}
```

### Main

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Main {
    public static void main(String[] args) {
        String username = "test";
        String password = "test";

        UserValidator ldapValidator = new UserValidator("LDAP");
        UserValidator dbValidator = new UserValidator("DataBase");

        TaskValidator ldapTask = new TaskValidator(ldapValidator, username, password);
        TaskValidator dbTask = new TaskValidator(dbValidator, username, password);


        List<TaskValidator> taskList = new ArrayList<>();
        taskList.add(ldapTask);
        taskList.add(dbTask);

        ExecutorService executor = (ExecutorService) Executors.newCachedThreadPool();

        String result;
        try {
            result = executor.invokeAny(taskList);
            System.out.printf("Main: Result: %s\n", result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        executor.shutdown();
    }
}
```

## Separating the launching of tasks and the processing of their results in an executor

Normally, when you execute concurrent tasks using an executor, you will send Runnable
or Callable tasks to the executor and get Future objects to control the method. You can find situations, where you need to send the tasks to the executor in one object and process
the results in another one. For such situations, the Java Concurrency API provides the
CompletionService class.


This CompletionService class has a method to send the tasks to an executor and a
method to get the Future object for the next task that has  nished its execution. Internally,
it uses an Executor object to execute the tasks. This behavior has the advantage to share a CompletionService object, and sends tasks to the executor so the others can process the results. The limitation is that the second object can only get the Future objects for those tasks that have  nished its execution, so these Future objects can only be used to get the

results of the tasks.


​			
​		
​	
​		
​	

### ReportGenerator

```java
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class ReportGenerator implements Callable<String> {
    private String sender;
    private String title;

    // implement the constructor of the class the initializes the two attributes
    public ReportGenerator(String sender, String title) {
        this.sender = sender;
        this.title = title;
    }

    @Override
    public String call() throws  Exception {
        try {
            Long duration = (long)(Math.random()*10);
            System.out.printf("%s_%s: ReportGenerator: Generating a report during %d seconds \n", this.sender, this.title, duration);
            TimeUnit.SECONDS.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // generate the report as a string with the sender and title attributes and return the string
        String ret = sender + ": " + title;
        return ret;
    }
}
```

### ReportRequest

```java
import java.util.concurrent.CompletionService;


// create a class named ReportRequest and specify that it implements the Runnable interface. This class will simulate some report request.
public class ReportRequest implements Runnable {
    private String name;
    private CompletionService<String> service;

    public ReportRequest(String name, CompletionService<String> service) {
        this.name = name;
        this.service = service;
    }

    //Implement the run() method. Create three ReportGenerator objects and send them to the CompletionService object using the submit() method.
    @Override
    public void run() {
        ReportGenerator reportGenerator = new ReportGenerator(name, "Report");
        service.submit(reportGenerator);
    }
}
```

### ReportProcessor

```java
import java.util.concurrent.CompletionService;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;


// this class will get the results of the ReportGenerator tasks
public class ReportProcessor implements Runnable {
    private CompletionService<String> service;
    private boolean end;

    public ReportProcessor(CompletionService<String> service) {
        this.service = service;
        end = false;
    }

    @Override
    public void run() {
        while(!end) {
            try {
                // poll() checks if there are any Future objects in the queue. if the  queue is empty, it returns null. Otherwise, it returns its first element and removes it from the queue.
                Future<String> result = service.poll(20, TimeUnit.SECONDS);
                if (result != null) {
                    String report = result.get();
                    System.out.printf("ReportReceiver: Report Received: %s\n", report);
                }
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
    }

    public void setEnd(boolean end) {
        this.end = end;
    }
}
```

### Main

```java
import java.util.concurrent.*;

public class Main {
    public static void main(String[] args) {
        // create ThreadExecutor using the newCachedThreadPool() method of the Executors class
        ExecutorService executor = (ExecutorService)Executors.newCachedThreadPool();

        // create CompletionService using the executor created earlier as a parameter of the constructor
        CompletionService<String> service = new ExecutorCompletionService<String>(executor);

        // create two reportrequest objects and the threads to execute them
        ReportRequest faceRequest = new ReportRequest("Face", service);
        ReportRequest onlineRequest = new ReportRequest("Online", service);
        Thread faceThread = new Thread(faceRequest);
        Thread onlineThread = new Thread(onlineRequest);

        ReportProcessor processor = new ReportProcessor(service);
        Thread senderThread = new Thread(processor);

        // start the three threads
        System.out.printf("Main: Starting the Threads");
        faceThread.start();
        onlineThread.start();
        senderThread.start();

        // wait for the finalization of the reportrequest threads;
        try {
            System.out.printf("Main: Waiting for the report generators");
            faceThread.join();
            onlineThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Main: Shutting down the executor.\n");
        executor.shutdown();
        try {
            executor.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // finish the execution of the reportsender object setting the value of its end attribute to true
        processor.setEnd(true);
        System.out.printf("Main: Ends");
    }
}
```

