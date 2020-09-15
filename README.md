### Hi there 👋

<!--
**jsycdut/jsycdut** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
大佬说，这题不是直接扫一遍就过了么？？？

我：。。。。。


*[_Gelly_]* *The main method caused an error: No result found for job, was execute() called before getting the result?*

I download [flink-1.11.1-bin-scala_2.12.tgz|http://apache.mirrors.pair.com/flink/flink-1.11.1/flink-1.11.1-bin-scala_2.12.tgz] from the official site of flink, then do as [Running Gelly Examples|https://ci.apa%20che.org/projects/flink/flink-docs-release-1.11/dev/libs/gelly/#running-gelly-examples] says to try the pagerank algorithm and hit the problem above,  the details are shown as below (you can reproduce the error if you follow the steps)
{code:bash}
➜ dist tar -xf flink-1.11.1-bin-scala_2.12.tgz
➜ dist cd flink-1.11.1
➜  flink-1.11.1 cp -v opt/flink-gelly*.jar lib  # it copies two gelly jars
'opt/flink-gelly_2.12-1.11.1.jar' -> 'lib/flink-gelly_2.12-1.11.1.jar'
'opt/flink-gelly-scala_2.12-1.11.1.jar' -> 'lib/flink-gelly-scala_2.12-1.11.1.jar'

 ➜ flink-1.11.1 ./bin/start-cluster.sh
 Starting cluster.
 Starting standalonesession daemon on host HP-280-Pro-G4-MT-Business-PC.
 Starting taskexecutor daemon on host HP-280-Pro-G4-MT-Business-PC.
 ➜ flink-1.11.1 ./bin/flink run examples/gelly/flink-gelly-examples_2.12-1.11.1.jar --algorithm PageRank --input StarGraph --vertex_count 5 --output Print
 Job has been submitted with JobID f867abf1d2cd94d07a419591e41b63a5
 Program execution finished
 Job with JobID f867abf1d2cd94d07a419591e41b63a5 has finished.
 Job Runtime: 1647 ms
 Accumulator Results:
 - 6907b5f63ee1f31af9715772ddcff154-collect (java.util.ArrayList) [5 elements]

 # ERROR messages show up 
 ------------------------------------------------------------
 The program finished with the following exception:

org.apache.flink.client.program.ProgramInvocationException: The main method caused an error: No result found for job, was execute() called before getting the result?
 at org.apache.flink.client.program.PackagedProgram.callMainMethod(PackagedProgram.java:302)
 at org.apache.flink.client.program.PackagedProgram.invokeInteractiveModeForExecution(PackagedProgram.java:198)
 at org.apache.flink.client.ClientUtils.executeProgram(ClientUtils.java:149)
 at org.apache.flink.client.cli.CliFrontend.executeProgram(CliFrontend.java:699)
 at org.apache.flink.client.cli.CliFrontend.run(CliFrontend.java:232)
 at org.apache.flink.client.cli.CliFrontend.parseParameters(CliFrontend.java:916)
 at org.apache.flink.client.cli.CliFrontend.lambda$main$10(CliFrontend.java:992)
 at org.apache.flink.runtime.security.contexts.NoOpSecurityContext.runSecured(NoOpSecurityContext.java:30)
 at org.apache.flink.client.cli.CliFrontend.main(CliFrontend.java:992)
 Caused by: java.lang.NullPointerException: No result found for job, was execute() called before getting the result?
 at org.apache.flink.util.Preconditions.checkNotNull(Preconditions.java:75)
 at org.apache.flink.graph.AnalyticHelper.getAccumulator(AnalyticHelper.java:81)
 at org.apache.flink.graph.asm.dataset.Collect.getResult(Collect.java:62)
 at org.apache.flink.graph.asm.dataset.Collect.getResult(Collect.java:35)
 at org.apache.flink.graph.asm.dataset.DataSetAnalyticBase.execute(DataSetAnalyticBase.java:56)
 at org.apache.flink.graph.drivers.output.Print.write(Print.java:48)
 at org.apache.flink.graph.Runner.execute(Runner.java:454)
 at org.apache.flink.graph.Runner.main(Runner.java:507)
 at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
 at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
 at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
 at java.lang.reflect.Method.invoke(Method.java:498)
 at org.apache.flink.client.program.PackagedProgram.callMainMethod(PackagedProgram.java:288)
 ... 8 more
{code}
 

I found the reason why flink goes wrong after debugging the code remotely, it's simply due to that ContextEnvironment doesn't assign the jobExecutionResult to lastJobExecutionResult after finishing the job and AnalyticHelper#getAccumulator will get the lastJobExecutionResult, finally things goes wrong.

the logic behind the scene is
{code:java}
// ContextEnvironment#execute(String jobName)
@Override
public JobExecutionResult execute(String jobName) throws Exception {
  final JobClient jobClient = executeAsync(jobName);
  final List<JobListener> jobListeners = getJobListeners();

  try {
    final JobExecutionResult  jobExecutionResult = getJobExecutionResult(jobClient);
    jobListeners.forEach(jobListener ->
    jobListener.onJobExecuted(jobExecutionResult, null));

    // the missing code
    // this.lastJobExecutionResult =  jobExecutionResult;
    return jobExecutionResult;
  } catch (Throwable t) {
    jobListeners.forEach(jobListener ->
        jobListener.onJobExecuted(null, ExceptionUtils.stripExecutionException(t)));
    ExceptionUtils.rethrowException(t);

    // never reached, only make javac happy
    return null;
  }
}

// where the error happens 
// AnalyticHelper#getAccumulator(ExecutionEnvironment env, String accumulatorName)
public <A> A getAccumulator(ExecutionEnvironment env, String accumulatorName) {
  // the result is null due to the missing assignment to lastJobExecutionResult in ContextEnvironment#execute(String jobName)
  JobExecutionResult result = env.getLastJobExecutionResult();

  // error raised here
  Preconditions.checkNotNull(result, "No result found for job, was execute() called before getting the result?");

  return result.getAccumulatorResult(id + SEPARATOR + accumulatorName);
}
{code}
I'd like to fix this problem, can anyone assign me a ticket?

 
