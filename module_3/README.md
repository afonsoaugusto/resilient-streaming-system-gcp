# Module 3: Implementing Streaming Pipelines

## Lab 2 : Building Resilient Streaming Systems Lab 2 : Streaming Data Pipelines

### Tarefa 1: preparação

```sh
gcloud beta compute --project "qwiklabs-gcp-fb8e935db314688c" ssh --zone "us-central1-a" "training-vm"
cp -r /training/training-data-analyst/ .
source /training/project_env.sh
```

```sh
google4703313_student@training-vm:~$ cat /training/project_env.sh
#! /bin/bash

# Create the DEVSHELL_PROJECT_ID on a VM
curl "http://metadata.google.internal/computeMetadata/v1/project/project-id" -H "Metadata-Flavor: Google" > Project_ID
awk '{print "export DEVSHELL_PROJECT_ID=" $0, "\n" "export BUCKET=" $0, "\n" "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre" }' Project_ID > env.txt
source env.txt
echo $DEVSHELL_PROJECT_ID
```

### Tarefa 2: crie um conjunto de dados do BigQuery e um intervalo do Cloud Storage

### Tarefa 3: simule dados do sensor de tráfego no Pub/Sub

```sh
/training/sensor_magic.sh
```

```sh
google4703313_student@training-vm:~$ cat /training/sensor_magic.sh
#! /bin/bash

# User tasks:
#  1. copy repo to ~/training-data-analyst
#  2. create $DEVSHELL_PROJECT_ID
#
# Install PIP
# sudo apt-get install -y python-pip
# Use PIP to install pubsub API
# sudo pip install -U google-cloud-pubsub
# Download the data file
gsutil cp gs://cloud-training-demos/sandiego/sensor_obs2008.csv.gz ~/training-data-analyst/courses/streaming/publish/
# cd to directory
cd ~/training-data-analyst/courses/streaming/publish/
# Run sensor simulator
python ./send_sensor_data.py --speedFactor=60 --project $DEVSHELL_PROJECT_ID
```

```sh
gcloud beta compute --project "qwiklabs-gcp-fb8e935db314688c" ssh --zone "us-central1-a" "training-vm"
source /training/project_env.sh
```

### Tarefa 4: inicie o pipeline do Dataflow

```sh
cd ~/training-data-analyst/courses/streaming/process/sandiego
cat run_oncloud.sh
```

```sh
cat run_oncloud.sh
#!/bin/bash

if [ "$#" -lt 3 ]; then
   echo "Usage:   ./run_oncloud.sh project-name bucket-name classname [options] "
   echo "Example: ./run_oncloud.sh cloud-training-demos cloud-training-demos CurrentConditions --bigtable"
   exit
fi

PROJECT=$1
shift
BUCKET=$1
shift
MAIN=com.google.cloud.training.dataanalyst.sandiego.$1
shift

echo "Launching $MAIN project=$PROJECT bucket=$BUCKET $*"

export PATH=/usr/lib/jvm/java-8-openjdk-amd64/bin/:$PATH
mvn compile -e exec:java \
 -Dexec.mainClass=$MAIN \
      -Dexec.args="--project=$PROJECT \
      --stagingLocation=gs://$BUCKET/staging/ $* \
      --tempLocation=gs://$BUCKET/staging/ \
      --runner=DataflowRunner"


# If you run into quota problems, add this option the command line above
#     --maxNumWorkers=2
# In this case, you will not be able to view autoscaling, however.

```

```sh
https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/streaming/process/sandiego/run_oncloud.sh
```

```sh
cd ~/training-data-analyst/courses/streaming/process/sandiego/src/main/java/com/google/cloud/training/dataanalyst/sandiego

cat AverageSpeeds.java
```

[AverageSpeeds.java](https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/streaming/process/sandiego/src/main/java/com/google/cloud/training/dataanalyst/sandiego/AverageSpeeds.java)
```java
/*
 * Copyright (C) 2016 Google Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not
 * use this file except in compliance with the License. You may obtain a copy of
 * the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations under
 * the License.
 */

package com.google.cloud.training.dataanalyst.sandiego;

import java.util.ArrayList;
import java.util.List;

import org.apache.beam.runners.dataflow.options.DataflowPipelineOptions;
import org.apache.beam.sdk.Pipeline;
import org.apache.beam.sdk.io.gcp.pubsub.PubsubIO;
import org.apache.beam.sdk.io.gcp.bigquery.BigQueryIO;
import org.apache.beam.sdk.options.Default;
import org.apache.beam.sdk.options.Description;
import org.apache.beam.sdk.options.PipelineOptionsFactory;
import org.apache.beam.sdk.transforms.DoFn;
import org.apache.beam.sdk.transforms.Mean;
import org.apache.beam.sdk.transforms.ParDo;
import org.apache.beam.sdk.transforms.windowing.SlidingWindows;
import org.apache.beam.sdk.transforms.windowing.Window;
import org.apache.beam.sdk.values.KV;
import org.apache.beam.sdk.values.PCollection;
import org.joda.time.Duration;
import org.joda.time.Instant;

import com.google.api.services.bigquery.model.TableFieldSchema;
import com.google.api.services.bigquery.model.TableRow;
import com.google.api.services.bigquery.model.TableSchema;

/**
 * A dataflow pipeline that computes average speeds in each lane
 * 
 * @author vlakshmanan
 *
 */
public class AverageSpeeds {

  public static interface MyOptions extends DataflowPipelineOptions {
    @Description("Over how long a time period should we average? (in minutes)")
    @Default.Double(60.0)
    Double getAveragingInterval();

    void setAveragingInterval(Double d);

    @Description("Simulation speedup factor. Use 1.0 if no speedup")
    @Default.Double(60.0)
    Double getSpeedupFactor();

    void setSpeedupFactor(Double d);
  }

  @SuppressWarnings("serial")
  public static void main(String[] args) {
    MyOptions options = PipelineOptionsFactory.fromArgs(args).withValidation().as(MyOptions.class);
    options.setStreaming(true);
    Pipeline p = Pipeline.create(options);

    String topic = "projects/" + options.getProject() + "/topics/sandiego";
    String avgSpeedTable = options.getProject() + ":demos.average_speeds";

    // if we need to average over 60 minutes and speedup is 30x
    // then we need to average over 2 minutes of sped-up stream
    Duration averagingInterval = Duration
        .millis(Math.round(1000 * 60 * (options.getAveragingInterval() / options.getSpeedupFactor())));
    Duration averagingFrequency = averagingInterval.dividedBy(2); // 2 times
    // in
    // window
    System.out.println("Averaging interval = " + averagingInterval);
    System.out.println("Averaging freq = " + averagingFrequency);

    // Build the table schema for the output table.
    List<TableFieldSchema> fields = new ArrayList<>();
    fields.add(new TableFieldSchema().setName("timestamp").setType("TIMESTAMP"));
    fields.add(new TableFieldSchema().setName("latitude").setType("FLOAT"));
    fields.add(new TableFieldSchema().setName("longitude").setType("FLOAT"));
    fields.add(new TableFieldSchema().setName("highway").setType("STRING"));
    fields.add(new TableFieldSchema().setName("direction").setType("STRING"));
    fields.add(new TableFieldSchema().setName("lane").setType("INTEGER"));
    fields.add(new TableFieldSchema().setName("speed").setType("FLOAT"));
    fields.add(new TableFieldSchema().setName("sensorId").setType("STRING"));
    TableSchema schema = new TableSchema().setFields(fields);

    PCollection<LaneInfo> currentConditions = p //
        .apply("GetMessages", PubsubIO.readStrings().fromTopic(topic)) //
        .apply("ExtractData", ParDo.of(new DoFn<String, LaneInfo>() {
          @ProcessElement
          public void processElement(ProcessContext c) throws Exception {
            String line = c.element();
            c.output(LaneInfo.newLaneInfo(line));
          }
        }));

    PCollection<KV<String, Double>> avgSpeed = currentConditions //
        .apply("TimeWindow",
            Window.into(SlidingWindows//
                .of(averagingInterval)//
                .every(averagingFrequency))) //
        .apply("BySensor", ParDo.of(new DoFn<LaneInfo, KV<String, Double>>() {
          @ProcessElement
          public void processElement(ProcessContext c) throws Exception {
            LaneInfo info = c.element();
            String key = info.getSensorKey();
            Double speed = info.getSpeed();
            c.output(KV.of(key, speed));
          }
        })) //
        .apply("AvgBySensor", Mean.perKey());

    avgSpeed.apply("ToBQRow", ParDo.of(new DoFn<KV<String, Double>, TableRow>() {
      @ProcessElement
      public void processElement(ProcessContext c) throws Exception {
        TableRow row = new TableRow();
        String stationKey = c.element().getKey();
        Double speed = c.element().getValue();
        String line = Instant.now().toString() + "," + stationKey + "," + speed; // CSV
        LaneInfo info = LaneInfo.newLaneInfo(line);
        row.set("timestamp", info.getTimestamp());
        row.set("latitude", info.getLatitude());
        row.set("longitude", info.getLongitude());
        row.set("highway", info.getHighway());
        row.set("direction", info.getDirection());
        row.set("lane", info.getLane());
        row.set("speed", info.getSpeed());
        row.set("sensorId", info.getSensorKey());
        c.output(row);
      }
    })) //
        .apply(BigQueryIO.writeTableRows().to(avgSpeedTable)//
            .withSchema(schema)//
            .withWriteDisposition(BigQueryIO.Write.WriteDisposition.WRITE_APPEND)
            .withCreateDisposition(BigQueryIO.Write.CreateDisposition.CREATE_IF_NEEDED));

    p.run();
  }
}
```

```sh
cd ~/training-data-analyst/courses/streaming/process/sandiego

./run_oncloud.sh $DEVSHELL_PROJECT_ID $BUCKET AverageSpeeds
```

```sql
SELECT *
FROM [<PROJECTID>:demos.average_speeds]
ORDER BY timestamp DESC
LIMIT 100
```

```sql
SELECT
  MAX(timestamp)
FROM
  [<PROJECTID>:demos.average_speeds]
```

```sql
SELECT
  *
FROM
  [<PROJECTID>:demos.average_speeds@-600000]
ORDER BY
  timestamp DESC
```

### Tarefa 8: observe e entenda o escalonamento automático

### Tarefa 9: atualize o script de simulação de dados do sensor

```sh
cd ~/training-data-analyst/courses/streaming/publish

./send_sensor_data.py --speedFactor=60 --project $DEVSHELL_PROJECT_ID
```

### Tarefa 10: integração do Stackdriver

### Tarefa 11: analise as métricas

### Tarefa 14: inicie outro pipeline de streaming

```sh
cd ~/training-data-analyst/courses/streaming/process/sandiego/src/main/java/com/google/cloud/training/dataanalyst/sandiego

cat CurrentConditions.java
```

[CurrentConditions.java](https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/streaming/process/sandiego/src/main/java/com/google/cloud/training/dataanalyst/sandiego/CurrentConditions.java)

```java
/*
 * Copyright (C) 2016 Google Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not
 * use this file except in compliance with the License. You may obtain a copy of
 * the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations under
 * the License.
 */

package com.google.cloud.training.dataanalyst.sandiego;

import java.util.ArrayList;
import java.util.List;

import org.apache.beam.runners.dataflow.options.DataflowPipelineOptions;
import org.apache.beam.sdk.Pipeline;
import org.apache.beam.sdk.io.gcp.bigquery.BigQueryIO;
import org.apache.beam.sdk.io.gcp.pubsub.PubsubIO;
import org.apache.beam.sdk.options.Default;
import org.apache.beam.sdk.options.Description;
import org.apache.beam.sdk.options.PipelineOptionsFactory;
import org.apache.beam.sdk.transforms.DoFn;
import org.apache.beam.sdk.transforms.ParDo;
import org.apache.beam.sdk.values.PCollection;

import com.google.api.services.bigquery.model.TableFieldSchema;
import com.google.api.services.bigquery.model.TableRow;
import com.google.api.services.bigquery.model.TableSchema;

/**
 * A dataflow pipeline that pulls from Pub/Sub and writes to BigQuery
 * 
 * @author vlakshmanan
 *
 */
@SuppressWarnings("serial")
public class CurrentConditions {

  public static interface MyOptions extends DataflowPipelineOptions {
    @Description("Also stream to Bigtable?")
    @Default.Boolean(false)
    boolean getBigtable();

    void setBigtable(boolean b);
  }

  public static void main(String[] args) {
    MyOptions options = PipelineOptionsFactory.fromArgs(args).withValidation().as(MyOptions.class);
    options.setStreaming(true);
    Pipeline p = Pipeline.create(options);

    String topic = "projects/" + options.getProject() + "/topics/sandiego";
    String currConditionsTable = options.getProject() + ":demos.current_conditions";

    // Build the table schema for the output table.
    List<TableFieldSchema> fields = new ArrayList<>();
    fields.add(new TableFieldSchema().setName("timestamp").setType("TIMESTAMP"));
    fields.add(new TableFieldSchema().setName("latitude").setType("FLOAT"));
    fields.add(new TableFieldSchema().setName("longitude").setType("FLOAT"));
    fields.add(new TableFieldSchema().setName("highway").setType("STRING"));
    fields.add(new TableFieldSchema().setName("direction").setType("STRING"));
    fields.add(new TableFieldSchema().setName("lane").setType("INTEGER"));
    fields.add(new TableFieldSchema().setName("speed").setType("FLOAT"));
    fields.add(new TableFieldSchema().setName("sensorId").setType("STRING"));
    TableSchema schema = new TableSchema().setFields(fields);

    PCollection<LaneInfo> currentConditions = p //
        .apply("GetMessages", PubsubIO.readStrings().fromTopic(topic)) //
        .apply("ExtractData", ParDo.of(new DoFn<String, LaneInfo>() {
          @ProcessElement
          public void processElement(ProcessContext c) throws Exception {
            String line = c.element();
            c.output(LaneInfo.newLaneInfo(line));
          }
        }));

    if (options.getBigtable()) {
      BigtableHelper.writeToBigtable(currentConditions, options);
    }

    currentConditions.apply("ToBQRow", ParDo.of(new DoFn<LaneInfo, TableRow>() {
      @ProcessElement
      public void processElement(ProcessContext c) throws Exception {
        TableRow row = new TableRow();
        LaneInfo info = c.element();
        row.set("timestamp", info.getTimestamp());
        row.set("latitude", info.getLatitude());
        row.set("longitude", info.getLongitude());
        row.set("highway", info.getHighway());
        row.set("direction", info.getDirection());
        row.set("lane", info.getLane());
        row.set("speed", info.getSpeed());
        row.set("sensorId", info.getSensorKey());
        c.output(row);
      }
    })) //
        .apply(BigQueryIO.writeTableRows().to(currConditionsTable)//
            .withSchema(schema)//
            .withWriteDisposition(BigQueryIO.Write.WriteDisposition.WRITE_APPEND)
            .withCreateDisposition(BigQueryIO.Write.CreateDisposition.CREATE_IF_NEEDED));

    p.run();
  }
}
```

## Module 3 Quiz

The Dataflow models provides constructs that map to the four questions that are relevant in any out-of-order data processing pipeline:


* 1. What results are calculated? B. Answered via transformations
* 2. Where in event time are results calculated? A. Answered via Event-time windowing
* 3. When in processing time are results materialized? D. Answered via Watermarks, triggers, and allowed lateness.
* 4. How do refinements of results relate?	C. Answered via Accumulation modes

```sh
```

```sh
```

```sh
```

```sh
```