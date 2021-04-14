# VM Build Getting Started for OS Harmonization

**Summary**:
This document assumes you are following the HL7 to FHIR Lab.

NOTE: This is the instalation for both
[the open source mapping engine](https://github.com/GoogleCloudPlatform/healthcare-data-harmonization)
and
[the Cloud DataFlow pipeline that invokes the engine.](https://github.com/GoogleCloudPlatform/healthcare-data-harmonization-dataflow)

### Pull the Git Repo

#### Step one

```shell
sudo apt install git
git clone https://github.com/GoogleCloudPlatform/healthcare-data-harmonization.git
git clone https://github.com/GoogleCloudPlatform/healthcare-data-harmonization-dataflow.git
```

### Install GCC Compiler

#### Step one

```shell
sudo apt update
sudo apt install build-essential
```

### Install Go Tools

#### Step one

```shell
sudo apt install curl
curl -O https://dl.google.com/go/go1.14.12.linux-amd64.tar.gz
```

#### Step two

```shell
tar xvf go1.14.12.linux-amd64.tar.gz
```

#### Step four

```shell
sudo chown -R root:root ./go
sudo mv go /usr/local
nano ~/.profile
```

Paste the following configuration to the bottom of the profile file and save.

```
export GOPATH=$HOME/work
export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin
```

#### Step five

```shell
source ~/.profile
mkdir $HOME/work
```

### Install Gradle

#### Step one

```shell
sudo apt install default-jdk
```

This may take a few minutes.

#### Step two

```shell
cd /usr/lib/jvm/
sudo mv java-11-openjdk-amd64 java-8-openjdk-amd64
```

#### Step three

```shell
cd /etc/alternatives
sudo ln -snf /usr/lib/jvm/java-8-openjdk-amd64/bin/java java
```

**Navigate back to your /home/username dir.**

#### Step four

```shell
sudo apt install wget
```

#### Step five

```shell
wget https://services.gradle.org/distributions/gradle-6.3-bin.zip -P /tmp
```

#### Step six

```shell
sudo apt install zip
```

#### Step seven

```shell
sudo unzip -d /opt/gradle /tmp/gradle-*.zip
```

#### Step eight

```shell
sudo nano /etc/profile.d/gradle.sh
```

Paste the following configuration to the bottom of the profile file and save.

```
export GRADLE_HOME=/opt/gradle/gradle-6.3
export PATH=${GRADLE_HOME}/bin:${PATH}
```

#### Step nine

```shell
sudo chmod +x /etc/profile.d/gradle.sh
source /etc/profile.d/gradle.sh
```

**Navigate back to your /home/username dir.**

### Install Protoc

#### Step one

```shell
wget https://github.com/google/protobuf/releases/download/v3.15.6/protobuf-all-3.15.6.tar.gz
```

#### Step two

```shell
tar xzf protobuf-all-3.15.6.tar.gz
```

#### Step three

```shell
cd protobuf-3.15.6
sudo apt-get install build-essential
```

#### Step four

```shell
sudo ./configure
```

#### Step six

```shell
sudo make
```

This will take several minutes.

#### Step seven

```shell
sudo make check
```

This could take up to 30 minutes.

#### Step eight

```shell
sudo make install
```

#### Step nine

```shell
sudo ldconfig
```

### Build the Mapping Engine

#### Step one

```shell
cd ../healthcare-data-harmonization/
./build_all.sh
```

This will take several minutes.

### Install the JAR file

#### Step one

```shell
cd ../healthcare-data-harmonization-dataflow/
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$PATH:/usr/bin/java
```

#### Step two

```shell
gradle wrapper --gradle-version 6.7.1
```

#### Step three

```shell
./gradlew shadowJar
```

### Create the Dataflow Pipeline

#### Step one

You will need to change the **project_id** variable below to be your project.

```shell
export PROJECT_ID=PROJECT_ID
```

```shell
export SUBSCRIPTION=hl7subscription
export ERROR_BUCKET=$PROJECT_ID/error
export MAPPING_BUCKET=$PROJECT_ID/mapping
export LOCATION=us-central1
export DATASET=datastore
export FHIRSTORE=fhirstore
```

#### Step two

```shell
java -jar build/libs/converter-0.1.0-all.jar --pubSubSubscription="projects/${PROJECT_ID}/subscriptions/${SUBSCRIPTION}" \
                                            --readErrorPath="gs://${ERROR_BUCKET}/read/read_error.txt" \
                                            --writeErrorPath="gs://${ERROR_BUCKET}/write/write_error.txt" \
                                            --mappingErrorPath="gs://${ERROR_BUCKET}/mapping/mapping_error.txt" \
                                            --mappingPath="gs://${MAPPING_BUCKET}/mapping_configs/hl7v2_fhir_r4/configurations/main.textproto" \
                                            --fhirStore="projects/${PROJECT}/locations/${LOCATION}/datasets/${DATASET}/fhirStores/${FHIRSTORE}" \
                                            --runner=DataflowRunner \
                                            --project=${PROJECT} \
                                            --region=$LOCATION
```

### Complete - return to the HL7v2 to FHIR Instructions.
