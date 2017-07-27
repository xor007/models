# configure log retention in EMR
set the following arguments when creating cluster:
```
Args=[\
"-y","yarn.log-aggregation-enable=true",\
"-y","yarn.log-aggregation.retain-seconds=43200",\
"-y","yarn.log-aggregation.retain-check-interval-seconds=1800",\
"-c","hadoop.http.staticuser.user=hadoop"\
] \
```


# fat jar through sbt for EMR

## build the dependencies
### install sbt
```
$ brew install sbt
```

### make sbt project

```
$ mkdir project
$ echo 'addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.11.2")' > project/plugins.sbt

$ cat <<'EOF' >build.sbt
name := "sqscli"

version := "1.0"

scalaVersion := "2.11.11"

import AssemblyKeys._  // put this at the top of the file

assemblySettings

libraryDependencies += "com.github.seratch" %% "awscala" % "0.6.+"
EOF
```

#### Hello Source file
```
cat <<'EOF' > sqscli.scala
import awscala._, sqs._

object HelloQ {
        def main(args: Array[String]) = {
        implicit val sqs = SQS.at(Region.EU_WEST_1)

        // create new queue
        val newQueueName = s"case2199509411-${System.currentTimeMillis}"
        val queue = sqs.createQueueAndReturnQueueName(newQueueName)
        val url = sqs.queueUrl(newQueueName)
        //log.info(s"Created queue: ${queue}, url: ${url}")

        queue.add("message body")
        queue.add("first", "second", "third")

        val messages: Seq[Message] = queue.messages
        queue.removeAll(messages)

        //queue.destroy()
    }
}
EOF
```

### test
```
$ sbt
[info] Loading project definition from ./sqscli/project
[info] Set current project to sqscli (in build file:./sqscli/)
> run
[info] Compiling 1 Scala source to ./sqscli/target/scala-2.11/classes...
[info] Running HelloQ
[success] Total time: 8 s, completed May 21, 2017 3:08:16 PM
```

### package
```
$ sbt assembly
...
[info] Packaging ./sqscli/target/scala-2.11/sqscli-assembly-1.0.jar ...
[info] Done packaging.
```

### copy to emr master node
```
$ scp target/scala-2.11/sqscli-assembly-1.0.jar  eu-mr:~/
```

### run in spark-shell
```
$ ssh eu-mr
[hadoop@ip-10-0-1-108 ~]$ spark-shell --master yarn --jars sqscli-assembly-1.0.jar
...
scala> import awscala._, sqs._
scala> implicit val sqs = SQS.at(Region.EU_WEST_1)
scala> val newQueueName = s"case2199509411-${System.currentTimeMillis}"
scala> val queue = sqs.createQueueAndReturnQueueName(newQueueName)
scala> val url = sqs.queueUrl(newQueueName)
scala> queue.add("message body")
scala> queue.add("first", "second", "third")
scala> val messages: Seq[Message] = queue.messages
```

You should see the queue in the sqs console

# delete older files in hdfs
### Delete older files on hdfs
#### save script bellow as dir_diff.sh

```
#!/bin/bash
usage="Usage: dir_diff.sh [base_dir] [day]"
if [ ! "$2" ]
then
  echo $usage
  exit 1
fi
base_dir=$1
day=$2

now=$(date +%s)
hadoop fs -ls -R $base_dir | grep "^d" | while read f; do
  dir_date=`echo $f | awk '{ print $6" "$7}'`
  difference=$(( ( $now - $(date -d "$dir_date" +%s) ) / ( 24 * 60 * 60 ) ))
  if [ $difference -gt $day ]; then
    echo $f;
  fi
done
```

#### find files older than 7 days and inspect them visually to be sure
```
./diff_hdfs.sh /var/log 7
```

### If sure, remove the files
```
./diff_hdfs.sh /var/log 7| awk '{print $8}' | while read f; do hadoop fs -rmr $f;done
```
