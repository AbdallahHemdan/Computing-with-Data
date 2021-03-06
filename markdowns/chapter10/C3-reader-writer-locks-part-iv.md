# Reader-Writer Locks - Part IV

Can we make it run faster? The answer — in this case — is a joyful yes;
we modify the program to use a reader-writer lock:

```java runnable
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class Shared {
  static final Map<String, int[]> cache = new HashMap<>();
  private static final ReadWriteLock lock = new ReentrantReadWriteLock();
  static final Lock readLock = lock.readLock();
  static final Lock writeLock = lock.writeLock();
}

class Writer extends Producer implements Runnable {
  public void run() {
    Shared.writeLock.lock();
    try {
      int[] referenceToValue = Shared.cache.get("key");
      
      for (int i = 0; i < referenceToValue.length; i++) {
        // Update value in-place; sum will add up to length
        referenceToValue[i] = produce();
      }
    } finally {
      Shared.writeLock.unlock();
    }
  }
}

class Reader extends Consumer implements Runnable {
  public void run() {
    Shared.readLock.lock();
    try {
      consume(Shared.cache.get("key"));
    } finally {
      Shared.readLock.unlock();
    }
  }
}

// { autofold
class Simulator {
  static void simulateWork() {
    try {
      Thread.sleep(3); // simulates work being done
    } catch (InterruptedException ex) {
      System.out.println(ex);
    }
  }
}

class Producer {
  protected int produce() {
    Simulator.simulateWork();
    return 1;
  }
}

class Consumer {
  protected void consume(int[] value) {
    int sum = Arrays.stream(value).sum();
    
    if (sum != 0 && sum != value.length) {
      throw new IllegalStateException("Partial sum: " + sum);
    }
    
    Simulator.simulateWork();
  }
}

public class Main {
  private static long runTrial() throws Exception {
    long startTime = System.nanoTime();
    final Thread[] threads = new Thread[1000];
    
    Shared.cache.put("key", new int[20]); // sum = 0 at this time
    threads[0] = new Thread(new Writer(), "Writer");
    threads[0].start();
    
    for (int i = 1; i < threads.length; i++) {
      threads[i] = new Thread(new Reader(), "Reader #" + i);
      threads[i].start();
    }
    
    for (int i = 1; i < threads.length; i++) {
      threads[i].join();
    }
    
    return System.nanoTime() - startTime;
  }
  
  public static void main(String args[]) throws Exception {
    long[] trials = new long[11];
    
    for (int i = 0; i < trials.length; i++) {
      trials[i] = runTrial();
    }
    
    Arrays.sort(trials);
    
    long median = trials[trials.length / 2];
    double average = Arrays.stream(trials).
      average().
      getAsDouble();
    
    System.out.printf(
      "Median time in nanoseconds: %1$,d\n",
      median);
    System.out.printf(
      "Average time in nanoseconds: %1$,.2f\n",
      average);
  }
}
// }
```
