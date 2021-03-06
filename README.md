# NPEDetector
NPEDetector is designed to find the potential null pointer exception in the systems writen by java(especially for distributed system). 
## Found and fixed bugs(total 26)
   [CLOUDSTACK-10356](https://issues.apache.org/jira/browse/CLOUDSTACK-10356)(11)
   [ZK-3006](https://issues.apache.org/jira/browse/ZOOKEEPER-3006)(1)
   [ZK-3007](https://issues.apache.org/jira/browse/ZOOKEEPER-3007)(1)
   [HBASE-20419](https://issues.apache.org/jira/browse/HBASE-20419)(2)
   [YARN-8164](https://issues.apache.org/jira/browse/YARN-8164)(3)
   [YARN-7786](https://issues.apache.org/jira/browse/YARN-7786)(1)
   [STORM-3048](https://github.com/apache/storm/pull/2657)(3)
   [ZK-3009](https://issues.apache.org/jira/browse/ZOOKEEPER-3009)(1)
   [ZK-3009-3.4](https://issues.apache.org/jira/browse/ZOOKEEPER-3009)(3)
## Found and confirmed bugs
   [HELIX-701](https://github.com/apache/helix/pull/200)(2)
   [ZK-3008](https://issues.apache.org/jira/browse/ZOOKEEPER-3008)(1)
   [HBASE-20420](https://issues.apache.org/jira/browse/HBASE-20420)(8)
   [HDFS-13451](https://issues.apache.org/jira/browse/HDFS-13451)(7)
   [STORM-3049](https://github.com/apache/storm/pull/2656)(2)
   [STORM-3051](https://github.com/apache/storm/pull/2656)(3)
## Found and pending bugs
   [HDFS-13452](https://issues.apache.org/jira/browse/HDFS-13452)(2)
   [CASSANDRA-14385](https://issues.apache.org/jira/browse/CASSANDRA-14385)(3)
   [ZK-3009](https://issues.apache.org/jira/browse/ZOOKEEPER-3009)(2)
   [ZK-3010](https://issues.apache.org/jira/browse/ZOOKEEPER-3010)(2)
   [ZK-3011](https://issues.apache.org/jira/browse/ZOOKEEPER-3011)(4)
   [STORM-3050](https://github.com/apache/storm/pull/2655)(1)
## False postive   
   [HELIX-702](https://github.com/apache/helix/pull/201)(3)
   [HBASE-20420](https://issues.apache.org/jira/browse/HBASE-20420)(1)
   [CLOUDSTACK-10356](https://issues.apache.org/jira/browse/CLOUDSTACK-10356)(1)
## Some great suggestions
   [ZK-3009](https://issues.apache.org/jira/browse/ZOOKEEPER-3009)
   [HDFS-13451](https://issues.apache.org/jira/browse/HDFS-13451)
# Motivation
<div  align="center">    
 <img src="https://github.com/lujiefsi/NPEDetector/blob/master/hbase-13546.png" width="60%" height="60%" alt="hbase-13546" align=center />
</div>

above figure shows the bug in hbase:

1. HMaster crash.
2. Zookeeper expire the connection, so data related to master is null.
3. Client send http request for get region server status before HMaster retoot
4. After receive the request, RS will get master data from Zookeeper
5.  Due to step 2, RS get null, and reference it w/o check it.

We can see that this bug is complex(involed 4 node and one crash event).
Actually, the developers have considered the master crash situation while parse:
<pre><code>
//callee: parse
public Master parse(byte[] data){
    if (data == null){
        return null;
    }
}
</code></pre>

but in its caller developer does not take the null pointer into account:
<pre><code>
//caller getMasterInfoPort
public getMasterInfoPort(byte[] data){
    Master master = parse(getData(false));
    return master.getInfoPort();
}
</code></pre>

This bug shows that NPE happends in corner case but some (callee) developers are wake up this 
case. So we develop NPEDetector to catch this simpe bug pattern:<font color=red size=4>callee return null, but caller
does not check it.</font>

# Approach
NPEDetector is based on an famous static analysis framework [WALA](https://github.com/wala/WALA).
We apply two analysis strategies in NPEDetector, difference in step 4:

    step1 : find all return null method(RNM)

    step2 : find all RNM' caller;

    step3 : find all RNM return value's use instruction.

    step4 : simple: check if null checker exists in caller, without construct ControldependencyGraph
	        complex:check all use instructions whether controled by check null condition(CNC)

    step5 : Score each callee:CNC numbers * 10 - caller number.

    step6 : Sort all callees and print.
Simple strategy may cause false negatives like:
 <pre><code>
    ret = foo();
    if (ret != null) ret.foo1;
    ret.field;//NPE, but our tool won't not reporte it.
</code></pre>
  
In step5, we score each callee based on:
+ if some developer have consider CNC, but some are not, we think no CNC developeres are wrong
+ developer may bother with those massive [CNC](https://stackoverflow.com/questions/271526/avoiding-null-statements/271874#271874)
    
# Usage
1. We use maven build our project, so you can use eclipse or other IDE import it as existed maven project. You can also use "mvn clean compile assembly:single" to generate a runnable jar, but need to do step 2 first.
2. vim the WALA  configuration file: ./NPEDetector/src/main/resources/wala.properties, you need to change:  

> 2.1 the property *java_runtime_dir* to your jre1.7 path.

> 2.2 set *jardir* as the jars path to be analyzed 

> 2.3 set *outputfile* as you want to dump result

> 2.4 set *debug* to false or true, this is for debug!
