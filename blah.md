---
layout: post
title: Visualizing Sensors
image: https://www.parisi.io/wp-content/uploads/2018/01/2738-04-300x225.jpg
---

<h1 style="text-align: center;">Visualizing Temperature and Humidity Sensors to monitor my furnace</h1>
<p>As I've previously discussed with 'Navigating IoT,' I think that IoT is such a general initialism that we can't fully capture what is possible. One of the primary areas technologists refer to is sensor capture. Unlike most posts that discuss this, I had an actual reason to capture the data. I have wireless sensors that read temperature and humidity; however, I don't have any way of capturing data from them. With the use of a <a href="https://www.raspberrypi.org/products/sense-hat/">SenseHAT</a> on a few well placed Raspberry PIs I was able to get this information to Grafana and visualize it very easily.
</p>
<figure class="caption">
<p >
<a href="https://www.adafruit.com/product/2738"><img class="wp-image-323 size-medium" src="https://www.parisi.io/wp-content/uploads/2018/01/2738-04-300x225.jpg" alt="" width="300" height="225" /><figcaption>SenseHAT on Raspberry PI</figcaption></a>
</p>
</figure>
<h2>Getting Started</h2>
<p>
This how-to will step you through setting up sensors on your Raspberry PI using Apache MiNiFi C++. My first step was to assemble the RPIs with SenseHATs. This is as simple as connecting the SenseHAT to the GPIO ports on the RPI [1].

I next built an image of Raspberry PI using a custom branch with some I2C capabilities [2]. This <a href="https://github.com/phrocker/nifi-minifi-cpp/tree/PiCar">branch</a> contains intermediate code used for a Raspberry PI driven car; however, it also contains a processor called SenseHAT. To get this up and running I included a third party library named <a href="https://github.com/RTIMULib/RTIMULib2">RTIMULIB2.</a> The reason I chose the SenseHAT is because I knew getting the processor running would be simple with this library. Alternatively, you could use ExecuteScript with a python script, but the responsiveness of the C++ calls was much higher and required very little code.

As you can see from the onSchedule and onTrigger functions, below, there is very little to getting this running. With this processor included I built MiNiFi on the PI and installed it. I bootstrapped the agent without execute script, lib archive, or expression language capabilities. I enabled my custom extension with cmake -DENABLE_I2C=true ..
</p>
{% highlight c++ linenos %}
void SenseHAT::onSchedule(const std::shared_ptr<core::ProcessContext> &context, const std::shared_ptr<core::ProcessSessionFactory&gt; &sessionFactory) {

  imu = RTIMU::createIMU(&settings);
  if (imu) {
    imu->IMUInit();
    imu->setGyroEnable(true);
    imu->setAccelEnable(true);
  } else {
    throw std::runtime_error("RTIMU could not be initialized");
  }

  humidity_sensor_ = RTHumidity::createHumidity(&settings);
  if (humidity_sensor_) {
    humidity_sensor_->humidityInit();
  } else {
    throw std::runtime_error("RTHumidity could not be initialized");
  }

  pressure_sensor_ = RTPressure::createPressure(&settings);
  if (pressure_sensor_) {
    pressure_sensor_->pressureInit();
  } else {
    throw std::runtime_error("RTPressure could not be initialized");
  }

}

void SenseHAT::onTrigger(const std::shared_ptr<core::ProcessContext> &context, const std::shared_ptr<core::ProcessSession&gt; &session) {

auto flow_file_ = session->create();
flow_file_->setSize(0);

if ( imu->IMURead() ){
  RTIMU_DATA imuData = imu->getIMUData();
  auto vector = imuData.accel;
  std::cout << "acceleration" << std::endl;
  std::string degrees = RTMath::displayDegrees("acceleration",vector);
  flow_file_->addAttribute("ACCELERATION", degrees);
}

RTIMU_DATA data;

bool have_sensor = false;

if (humidity_sensor_->humidityRead(data)) {
  if (data.humidityValid) {
    have_sensor = true;
    std::stringstream ss;
    ss << std::fixed << std::setprecision(2) << data.humidity;
    flow_file_->addAttribute("HUMIDITY", ss.str());
  }
}

if (pressure_sensor_->pressureRead(data)) {
  if (data.pressureValid) {
    have_sensor = true;
    {
      std::stringstream ss;
      ss << std::fixed << std::setprecision(2) << data.pressure;
      flow_file_->addAttribute("PRESSURE", ss.str());
    }

    if (data.temperatureValid){
      std::stringstream ss;
      ss << std::fixed << std::setprecision(2) << data.temperature;
      flow_file_->addAttribute("TEMPERATURE", ss.str());
    }

  }
}

if (have_sensor) {

  WriteCallback callback("SenseHAT");

  session->write(flow_file_,&callback);
  session->transfer(flow_file_, Success);
}
{% endhighlight %}


<p>
I installed the MiNiFi agent in the root PI directory under ~/deploy/bin/ Once installed I created a flow that moved flow files created from the SenseHat Processor directly to a NiFi Instance through site to site. The NiFi instance is located on AWS, so all PIs could send data to it, using the s2s.host attribute as a differentiator. Note that Once I created the PI's image, I copied the SD card to three others where I placed them around my house. My goal was to get temperature and humidity readings in certain places. One in my basement, one upstairs in a utility room containing the furnace and furnace probe, and one in my office.

I would convert these Attributes to JSON and pass them along to InfluxDB by way of MQTT and Mosquitto. I used a similar setup to that found in this <a href="https://larsbergqvist.wordpress.com/2017/03/02/influxdb-and-grafana-for-sensor-time-series/">guide</a> [3]. The image, below, depicts the movement of data from SiteToSite to PublishMQTT. I then used the python script found in the guide as a framework for my own.

<img class="wp-image-331 size-large" src="https://www.parisi.io/wp-content/uploads/2018/01/Screen-Shot-2018-01-08-at-9.15.35-AM-1024x360.png" alt="" width="1024" height="360" /> Site2Site to InfluxDB

&nbsp;

My variance of the guide's python script is below. I ran this in the background, collecting data from mosquitto and inserting it into InfluxDB.
Grafana can use InfluxDB as a data source, querying the appropriate fields as necessary. When finished I found the mean humidity and temperature to be higher than expected. The reason was an outlier in my office. All temperatures on the SenseHAT register warmer than the ambient temperature due to the heat of the processor, below; however, the one in my office registered much higher due to it being on the highest level and it is the room with the poorest airflow. The temperatures are in celcius. Note that there is an overall increase throughout the night. This is because my furnace runs longer as the outdoor temperature decreases. The stark rise is when the furnace is running followed by a drop as the thermostat temperature acquiesces. Humidity rises and lowers as the whole house humidifier is running concurrently with the furnace.
</p>
&nbsp;

{% highlight python linenos %}
#!/usr/bin/env python3
import paho.mqtt.client as mqtt
import datetime
import time
import json
from influxdb import InfluxDBClient

def on_connect(client, userdata, flags, rc):
    client.subscribe("sensors")

def on_message(client, userdata, msg):
    # Use utc as timestamp
    print("oh i got something")
    receiveTime=datetime.datetime.utcnow()
    message=msg.payload.decode("utf-8")
    parsedJson=False
    try:
        val  = json.loads(message)
        parsedJson=True
        print("good json")
    except:
        parsedJson=False

    if parsedJson:
        json_body = [
            {
                "measurement": "temperature",
                "time": receiveTime,
                "fields": {
                    "temperature": float(val['TEMPERATURE']),
                    "s2s.host": val['s2s.host']
                }
            }
        ]

        dbclient.write_points(json_body)

        json_body = [
            {
                "measurement": "humidity",
                "time": receiveTime,
                "fields": {
                    "humidity": float(val['HUMIDITY']),
                    "s2s.host": val['s2s.host']
                }
            }
        ]

        dbclient.write_points(json_body)

# Set up a client for InfluxDB
dbclient = InfluxDBClient('localhost', 8086, 'root', 'root', 'sensors')

# Initialize the MQTT client that should connect to the Mosquitto broker
client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message
connOK=False
while(connOK == False):
    try:
        client.connect("localhost", 1883, 60)
        connOK = True
    except:
        connOK = False
    time.sleep(2)

# Blocking loop to the Mosquitto broker
client.loop_forever()
{% endhighlight %}
&nbsp;

<img class="wp-image-333 size-large" src="https://www.parisi.io/wp-content/uploads/2018/01/Screen-Shot-2018-01-08-at-6.18.32-AM-1024x170.png" alt="" width="1024" height="170" /> Office Temp and Humidity

<p>I learned that there was only a minor difference in air temperature. This difference generally amounted to 1-2 degrees Fahrenheit, but this is enough to be felt. The basement is obviously cooler. Humidity in the basement wasn't higher, but this will likely be more stark in the late spring when the A/C isn't running. The above grade levels have lower humidity but higher temperatures.

With only a little bit of effort, I was able to capture and visualize temperature differentials and capture statistics. Using Apache MiNiFi C++, I was able to capture the SenseHAT and send the data to NiFi where I could do what I needed. The reason I chose MiNiFi over a simple python script that connected to a remote Mosquitto instance is primarily of provenance and controllability. With Apache MiNiFi I can control the agent through C&amp;C ( <strong>Command and Control</strong> ) capabilities. This will be especially useful as I can use command and control interfaces to update a flow on the agents when needed. Command and control became especially useful when I was running into heating issues on my office sensor and had to change the run time characteristics of the SenseHAT processor to run less often. I could do this remotely without any downtime. Since the agents were self registering with C&amp;C, this also meant doing so without having to SSH into the RPI. I'll go into further detail regarding my usage of command and control in a subsequent blog post.<p>

<p>
[1] <a href="https://www.adafruit.com/product/2738">https://www.adafruit.com/product/2738</a>
</p>

<p>
[2] <a href="https://github.com/phrocker/nifi-minifi-cpp/tree/PiCar">https://github.com/phrocker/nifi-minifi-cpp/tree/PiCar</a>
</p>
<p>
[3] <a href="https://larsbergqvist.wordpress.com/2017/03/02/influxdb-and-grafana-for-sensor-time-series/">https://larsbergqvist.wordpress.com/2017/03/02/influxdb-and-grafana-for-sensor-time-series/</a>
</p>
