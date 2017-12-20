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

