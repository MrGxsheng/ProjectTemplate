### 依赖

```xml

        <!--jmh 基准测试 -->
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>1.23</version>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>1.23</version>
            <scope>provided</scope>
        </dependency>

```

### test(以stringAdd 和 builder为例)

```java
package test;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

/**
 * JMH 基准测试
 *
 * <p>
 * fw练习生
 * </p>
 *
 * @author sheng
 * @since 2023/3/15 15:21:33
 */
@BenchmarkMode(Mode.AverageTime)
@State(Scope.Thread)
@Fork(1)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class JmhHello {
    String s = "";
    StringBuilder stringBuilder = new StringBuilder();

    @Benchmark
    public String stringAdd() {
        for (int i = 0; i < 1000; i++) s += i;
        return s;
    }

    @Benchmark
    public String stringBuilderAppend() {
        for (int i = 0; i < 1000; i++)
            stringBuilder.append(i);
        return stringBuilder.toString();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JmhHello.class.getSimpleName())
                .build();
        new Runner(opt).run();
    }
}

```

### 结果

```java
// 开始测试 stringAdd 方法
# JMH version: 1.23
# VM version: JDK 1.8.0_181, Java HotSpot(TM) 64-Bit Server VM, 25.181-b13
# VM invoker: D:\develop\Java\jdk8_181\jre\bin\java.exe
# VM options: -javaagent:C:\ideaIU-2020.1.3.win\lib\idea_rt.jar=50363:C:\ideaIU-2020.1.3.win\bin -Dfile.encoding=UTF-8
# Warmup: 3 iterations, 10 s each  // 预热运行三次
# Measurement: 5 iterations, 10 s each // 性能测试5次 
# Timeout: 10 min per iteration  // 超时时间10分钟
# Threads: 1 thread, will synchronize iterations  // 线程数量为1
# Benchmark mode: Average time, time/op  // 统计方法调用一次的平均时间
# Benchmark: net.codingme.jmh.JmhHello.stringAdd // 本次执行的方法

# Run progress: 0.00% complete, ETA 00:02:40
# Fork: 1 of 1
# Warmup Iteration   1: 95.153 ms/op  // 第一次预热，耗时95ms
# Warmup Iteration   2: 108.927 ms/op // 第二次预热，耗时108ms
# Warmup Iteration   3: 167.760 ms/op // 第三次预热，耗时167ms
Iteration   1: 198.897 ms/op  // 执行五次耗时度量
Iteration   2: 243.437 ms/op
Iteration   3: 271.171 ms/op
Iteration   4: 295.636 ms/op
Iteration   5: 327.822 ms/op


Result "net.codingme.jmh.JmhHello.stringAdd":
  267.393 ±(99.9%) 189.907 ms/op [Average]
  (min, avg, max) = (198.897, 267.393, 327.822), stdev = 49.318  // 执行的最小、平均、最大、误差值
  CI (99.9%): [77.486, 457.299] (assumes normal distribution)
  
// 开始测试 stringBuilderAppend 方法
# Benchmark: net.codingme.jmh.JmhHello.stringBuilderAppend

# Run progress: 50.00% complete, ETA 00:01:21
# Fork: 1 of 1
# Warmup Iteration   1: 1.872 ms/op
# Warmup Iteration   2: 4.491 ms/op
# Warmup Iteration   3: 5.866 ms/op
Iteration   1: 6.936 ms/op
Iteration   2: 8.465 ms/op
Iteration   3: 8.925 ms/op
Iteration   4: 9.766 ms/op
Iteration   5: 10.143 ms/op


Result "net.codingme.jmh.JmhHello.stringBuilderAppend":
  8.847 ±(99.9%) 4.844 ms/op [Average]
  (min, avg, max) = (6.936, 8.847, 10.143), stdev = 1.258
  CI (99.9%): [4.003, 13.691] (assumes normal distribution)


# Run complete. Total time: 00:02:42

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.
// 测试结果对比
Benchmark                     Mode  Cnt    Score     Error  Units
JmhHello.stringAdd            avgt    5  267.393 ± 189.907  ms/op
JmhHello.stringBuilderAppend  avgt    5    8.847 ±   4.844  ms/op

Process finished with exit code 0

```

### 注解

```java
@BenchmarkMode(Mode.AverageTime) //统计平均响应时间，不仅可以用在类上，也可用在测试方法上
@State(Scope.Thread) //每个进行基准测试的线程都会独享一个对象示例
@Fork(1) //每个进行基准测试的线程都会独享一个对象示例。
@OutputTimeUnit(TimeUnit.MILLISECONDS)  //进行 5 次微基准测试，也可用在测试方法上。
@Warmup(iterations = 3) //每个进行基准测试的线程都会独享一个对象示例。
@Measurement(iterations = 5) //进行 5 次微基准测试，也可用在测试方法上。

```

### 指定在main

此时main指定的优先级最高

```java
public static void main(String[] args) throws RunnerException {
    Options opt = new OptionsBuilder()
            .include(JmhHello.class.getSimpleName())
            .forks(1)
            .warmupIterations(5)
            .measurementIterations(10)
            .build();
    new Runner(opt).run();
}

```

