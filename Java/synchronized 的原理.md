# `synchronized` 的原理

在 Java 和 Kotlin 中，**`synchronized`** 是一种用于实现 **线程安全** 的关键字。它用于控制多个线程对共享资源的并发访问，确保在同一时间只有一个线程可以访问被保护的代码块或方法。

#### **1. `synchronized` 的基础用法**

```
java复制代码public class SynchronizedExample {
    private int counter = 0;

    public synchronized void increment() {
        counter++;
    }

    public synchronized int getCounter() {
        return counter;
    }
}
```

#### **解释：**

- `increment()` 和 `getCounter()` 方法使用了 `synchronized` 关键字，表示这两个方法在同一时间只能被一个线程访问。
- 这可以防止多个线程同时修改 `counter`，导致竞态条件（Race Condition）。

## **2. `synchronized` 的工作原理**

`Java` 中的 `synchronized` 关键字通过 **内置锁（Intrinsic Lock）** 或 **监视器锁（Monitor Lock）** 实现。这种锁是每个对象（`Object`）都隐含持有的。

#### **工作流程：**

- 当线程进入 `synchronized` 块或方法时，会尝试获取对象的 **Monitor 锁**。
- 如果锁被另一个线程持有，当前线程会被阻塞，直到持有锁的线程释放锁。
- 线程持有锁时，其他线程无法进入同一个锁保护的代码块。
- 当线程执行完 `synchronized` 块或方法后，会释放锁，使其他线程可以继续获取锁并执行。

#### **锁的粒度：**

- **对象级锁（Object Lock）**：`synchronized` 修饰的实例方法或代码块，锁是 **当前对象实例**。
- **类级锁（Class Lock）**：`synchronized` 修饰的静态方法或代码块，锁是 **当前类对象**（`Class` 对象）。

#### **示例：对象级锁与类级锁**

```
java复制代码public class SynchronizedDemo {
    public synchronized void instanceMethod() {
        System.out.println("Object-level lock");
    }

    public static synchronized void staticMethod() {
        System.out.println("Class-level lock");
    }
}
```

- `instanceMethod()` 使用的是对象级锁，即锁的是当前对象实例。
- `staticMethod()` 使用的是类级锁，即锁的是当前类的 `Class` 对象。

## **3. `synchronized` 的底层实现**

在字节码层面，`synchronized` 关键字会被转换为 **`monitorenter`** 和 **`monitorexit`** 指令。

#### **字节码示例：**

```
java复制代码public void synchronizedMethod() {
    synchronized (this) {
        System.out.println("Inside synchronized block");
    }
}
```

使用 `javap -c` 命令查看字节码：

```
plaintext复制代码0: aload_0
1: dup
2: monitorenter
3: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
6: ldc           #3                  // String Inside synchronized block
8: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
11: aload_0
12: monitorexit
13: goto          21
16: astore_1
17: aload_0
18: monitorexit
19: aload_1
20: athrow
21: return
```

#### **解释：**

- **`monitorenter`**：表示线程尝试获取锁。如果成功，进入 `synchronized` 块；否则，线程被阻塞。
- **`monitorexit`**：表示线程释放锁。
- 在异常情况（如抛出异常时），也会执行 `monitorexit`，确保锁被释放，避免死锁。

## **4. `synchronized` 的锁优化**

Java 虚拟机（JVM）在 **JDK 1.6** 之后，对 `synchronized` 做了多种优化，以提升性能：

#### **锁优化策略：**

1. **偏向锁（Biased Locking）**：
   - 如果一个锁仅被一个线程持有，JVM 会将锁标记为 **偏向锁**，避免频繁的 CAS 操作。
   - 偏向锁会在没有竞争的情况下自动升级为轻量级锁或重量级锁。
2. **轻量级锁（Lightweight Locking）**：
   - 当锁有竞争时，JVM 会将锁升级为轻量级锁，使用 **CAS 操作** 尝试获取锁。
   - 轻量级锁使用自旋等待，避免线程上下文切换的开销。
3. **重量级锁（Heavyweight Locking）**：
   - 当竞争变得激烈时，JVM 会升级为重量级锁，线程会被阻塞，避免 CPU 自旋过多消耗。
   - 重量级锁会使用操作系统的 **Mutex** 实现，效率较低。

## **5. `synchronized` 的优缺点**

#### **优点：**

- **简单易用**：`synchronized` 是 Java 提供的内置同步机制，使用简单，语义明确。
- **线程安全**：通过内置锁机制，保证了代码块或方法的线程安全。

#### **缺点：**

- **性能开销**：在高并发场景下，`synchronized` 的锁竞争可能导致性能下降。
- **阻塞线程**：重量级锁会阻塞线程，导致线程切换和上下文切换的开销。
- **锁升级与降级**：频繁的锁升级和降级操作可能导致性能波动。

## **6. `synchronized` 与 ReentrantLock 的对比**

| 特性         | `synchronized`                   | `ReentrantLock`                  |
| ------------ | -------------------------------- | -------------------------------- |
| 锁的类型     | 内置锁（Monitor）                | 显式锁（`java.util.concurrent`） |
| 锁的可重入性 | 支持                             | 支持                             |
| 可中断性     | 不支持                           | 支持                             |
| 锁超时       | 不支持                           | 支持（`tryLock()` 方法）         |
| 性能优化     | JVM 自动优化（偏向锁、轻量级锁） | 手动控制锁的释放与性能           |

#### **示例：使用 `ReentrantLock` 实现同步**

```java
import java.util.concurrent.locks.ReentrantLock;

public class LockExample {
    private final ReentrantLock lock = new ReentrantLock();
    private int counter = 0;

    public void increment() {
        lock.lock();
        try {
            counter++;
        } finally {
            lock.unlock();
        }
    }

    public int getCounter() {
        return counter;
    }
}
```

#### **解释：**

- `lock.lock()` 获取锁，`lock.unlock()` 释放锁，确保了线程安全。
- `ReentrantLock` 提供了更多功能，如可中断性和超时等待等。

## **总结**

- **`synchronized`** 是 Java 中的内置同步机制，使用内置锁（Monitor Lock）来保证线程安全。
- 底层通过字节码指令 `monitorenter` 和 `monitorexit` 实现。
- JVM 在 JDK 1.6 之后对 `synchronized` 进行了优化，使用偏向锁、轻量级锁和重量级锁来提升性能。
- 在高并发场景下，可以考虑使用 **`ReentrantLock`**，提供更多的功能和控制能力。
- 理解 `synchronized` 的工作原理和锁优化机制，可以帮助你编写更加高效和线程安全的并发程序。