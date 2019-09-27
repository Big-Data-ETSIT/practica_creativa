# Práctica de Big Data 2019 MUIT/MUIRST

The files and instructions for this lab were mainly extracted from the book [Agile Data Science 2.0](http://shop.oreilly.com/product/0636920051619.do), O'Reilly 2017. Now available at the [O'Reilly Store](http://shop.oreilly.com/product/0636920051619.do), on [Amazon](https://www.amazon.com/Agile-Data-Science-2-0-Applications/dp/1491960116) (in Paperback and Kindle) and on [O'Reilly Safari](https://www.safaribooksonline.com/library/view/agile-data-science/9781491960103/).
The original code is hosted in [this repository](https://github.com/rjurney/Agile_Data_Code_2)

# System Architecture

The following diagrams are pulled from the book, and express the basic concepts in the system architecture. The front and back end architectures work together to make a complete predictive system.

## Front End Architecture

This diagram shows how the front end architecture works in our flight delay prediction application. The user fills out a form with some basic information in a form on a web page, which is submitted to the server. The server fills out some neccesary fields derived from those in the form like "day of year" and emits a Kafka message containing a prediction request. Spark Streaming is listening on a Kafka queue for these requests, and makes the prediction, storing the result in MongoDB. Meanwhile, the client has received a UUID in the form's response, and has been polling another endpoint every second. Once the data is available in Mongo, the client's next request picks it up. Finally, the client displays the result of the prediction to the user! 

This setup is extremely fun to setup, operate and watch. Check out chapters 7 and 8 for more information!

![Front End Architecture](images/front_end_realtime_architecture.png)

## Back End Architecture

The back end architecture diagram shows how we train a classifier model using historical data (all flights from 2015) on disk (HDFS or Amazon S3, etc.) to predict flight delays in batch in Spark. We save the model to disk when it is ready. Next, we launch Zookeeper and a Kafka queue. We use Spark Streaming to load the classifier model, and then listen for prediction requests in a Kafka queue. When a prediction request arrives, Spark Streaming makes the prediction, storing the result in MongoDB where the web application can pick it up.

This architecture is extremely powerful, and it is a huge benefit that we get to use the same code in batch and in realtime with PySpark Streaming.

![Backend Architecture](images/back_end_realtime_architecture.png)

# Screenshots

Below are some examples of parts of the application we build in this book and in this repo. Check out the book for more!

## Airline Entity Page

Each airline gets its own entity page, complete with a summary of its fleet and a description pulled from Wikipedia.

![Airline Page](images/airline_page_enriched_wikipedia.png)

## Airplane Fleet Page

We demonstrate summarizing an entity with an airplane fleet page which describes the entire fleet.

![Airplane Fleet Page](images/airplanes_page_chart_v1_v2.png)

## Flight Delay Prediction UI

We create an entire realtime predictive system with a web front-end to submit prediction requests.

![Predicting Flight Delays UI](images/predicting_flight_kafka_waiting.png)

# Setting up the scenario
## Downloading the code
First, clone this repository by running the following command:

```
git clone https://github.com/ging/practica_big_data_2019
cd practica_big_data_2019
```

## Downloading Data

Now it is time to download the flight historical data:

```
resources/download_data.sh
```
## Installation

It is necessary to install quite a few dependencies in order to run all the components in the architecture. 
The following list includes some links to the installation procedure for each component:

 - [IntelliJ](https://www.jetbrains.com/help/idea/installation-guide.html) (:warning: Be sure to add Java JDK 1.8)
 - [Pyhton3](https://realpython.com/installing-python/) 
 - [pip](https://pip.pypa.io/en/stable/installing/)
 - [sbt](https://www.scala-sbt.org/release/docs/Setup.html) 
 - [MongoDB](https://docs.mongodb.com/manual/installation/)
 - [Apacha Spark](https://spark.apache.org/docs/latest/) (Suggested version 2.4.4)
 - [Apacha Zookeeper](https://zookeeper.apache.org/releases.html)
 - [Apacha Kafka](https://kafka.apache.org/quickstart) (Suggested version kafka_2.12-2.3.0)

### Install python dependencies

```
pip install requirements.txt
```
### Start Zookeeper

Open a console and go to the directory where you downloaded Zookeeper and run:

```
bin/zookeeper-server-start.sh config/zookeeper.properties
```
### Start Kafka

Open a console and go to the directory where you downloaded Kafka and run:

```
bin/kafka-server-start.sh config/server.properties
```

Open a new console in the same directory and create a new Kafka topic :
```
bin/kafka-topics.sh \
    --create \
   --zookeeper localhost:2181 \
    --replication-factor 1 \
    --partitions 1 \
    --topic flight_delay_classification_request
```
You should see the following message:
```
Created topic "flight_delay_classification_request".
```

You can see the topic we created with the `list topics` command:
```
bin/kafka-topics.sh --list --zookeeper localhost:2181
```
Output:
```
flight_delay_classification_request
```
(Optional) You can open a new console with a consumer in order to see the messages sent to that topic
```
bin/kafka-console-consumer.sh \
   --bootstrap-server localhost:9092 \
   --topic flight_delay_classification_request \
   --from-beginning
```

## Train and Save de the model with PySpark mllib
Now it is time to train the prediction model with the historical data. In a terminal, go to the base directory of the cloned repo, then access to the `resources` directory

```
cd practica_big_data_2019/resources/
```

Now, execute the script `train_spark_mllib_model.py`, which will perform all the training process for us. If you inspect the code, you will see that it reads the data we downloaded in the `data` folder and creates a model using the RandomForestClassifier algorithm from Spark.

```
python3 train_spark_mllib_model.py
```

As result of executing the script, the trained model will be saved in the `models` folder 

```
ls ../models
```   
## Run Flight Predictor
The flight predictor job will receive flight information through Kafka and, using Spark Streaming and the model we created, will make a prediction for the delay of said flight. You need to modify the `base_path` variable in the MakePrediction scala class (`/flight_prediction/src/main/scala/es/upm/dit/ging/predictor/MakePrediction.scala`) with the path of the directory where you have cloned the repository:
```
val base_path = "/home/student/practica_big_data_2019"
``` 
Then run this class using IntelliJ or Spark Submit. The latter needs some [extra configuration](https://spark.apache.org/docs/latest/submitting-applications.html). Examine the code and try to understand what is going on.


## Start the prediction request Web Application

In order to start the web application you need to access the `web` directory under `resources` and execute the Flask web application file `predict_flask.py`:
```
cd resources/web
python3 predict_flask.py
  
  ```
Now, visit `http://localhost:5000/flights/delays/predict_kafka` and open the JavaScript console. Using the web form provided, enter a nonzero departure delay, an ISO-formatted date (I used 2016-12-25, which was in the future at the time I was writing this), a valid carrier code (use AA or DL if you don’t know one), an origin and destination (my favorite is ATL → SFO), and a valid flight number (e.g., 1519), and hit Submit. Watch the debug output in the JavaScript console as the client polls for data from the response endpoint at `/flights/delays/predict/classify_realtime/response/`.
  
Quickly switch windows to your Spark console. Within 10 seconds (the length we have configured of a minibatch), you should see the predicted delay in the console, and also on the web interface.
  
## Check the predictions records inserted in MongoDB
```
$ mongo
> use use agile_data_science;
>db.flight_delay_classification_response.find();

```
You must have a similar output to the following:

```
{ "_id" : ObjectId("5d8dcb105e8b5622696d6f2e"), "Origin" : "ATL", "DayOfWeek" : 6, "DayOfYear" : 360, "DayOfMonth" : 25, "Dest" : "SFO", "DepDelay" : 290, "Timestamp" : ISODate("2019-09-27T08:40:48.175Z"), "FlightDate" : ISODate("2016-12-24T23:00:00Z"), "Carrier" : "AA", "UUID" : "8e90da7e-63f5-45f9-8f3d-7d948120e5a2", "Distance" : 2139, "Route" : "ATL-SFO", "Prediction" : 3 }
  { "_id" : ObjectId("5d8dcba85e8b562d1d0f9cb8"), "Origin" : "ATL", "DayOfWeek" : 6, "DayOfYear" : 360, "DayOfMonth" : 25, "Dest" : "SFO", "DepDelay" : 291, "Timestamp" : ISODate("2019-09-27T08:43:20.222Z"), "FlightDate" : ISODate("2016-12-24T23:00:00Z"), "Carrier" : "AA", "UUID" : "d3e44ea5-d42c-4874-b5f7-e8a62b006176", "Distance" : 2139, "Route" : "ATL-SFO", "Prediction" : 3 }
  { "_id" : ObjectId("5d8dcbe05e8b562d1d0f9cba"), "Origin" : "ATL", "DayOfWeek" : 6, "DayOfYear" : 360, "DayOfMonth" : 25, "Dest" : "SFO", "DepDelay" : 5, "Timestamp" : ISODate("2019-09-27T08:44:16.432Z"), "FlightDate" : ISODate("2016-12-24T23:00:00Z"), "Carrier" : "AA", "UUID" : "a153dfb1-172d-4232-819c-8f3687af8600", "Distance" : 2139, "Route" : "ATL-SFO", "Prediction" : 1 }


```
# Next steps

*  Try to use a docker container for each service in the architecture (Spark Master & Slave, Flask, Mongo, Kafka...)
*  Deploy the whole architecture using `docker-compose`
*  Deploy the whole architecture using Kubernetes
*  Deploy the whole architecture on Google Cloud/AWS
*  Change MongoDB for Cassandra
