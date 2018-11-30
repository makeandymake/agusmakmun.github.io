---
layout: post
title:  "Networked Nightlights with ESP8266 and Raspberry Pi"
date:   2018-11-29 21:00:00 +0100
---

_In the interest of clarity, I just want to prefix this post with a little disclaimer. This isn't an indepth tutorial, more a project log to inspire others that want to build similar projects. There's a minimum amount of knowledge required if you want to build this. You'll need to know how to add code in the Arduino IDE and upload it to your device and you'll also need some experience operating and connecting to a Raspberry Pi on the command line The code I've supplied is pretty basic. I'm not a full time software developer and the languages used here are certainly not my strong suite so be prepared for some hacky code that could probably be drastically improved. If you see any obvious room for improvement, I would love to hear about it!_

<img src="/static/img/nightlight.jpg">
  
A couple of years ago I made my oldest kid a nightlight for his bedroom. It was a simple project using a Raspberry Pi Zero, a Pimoroni Unicorn HAT and some code in Python. I set it up to change colours at specific times of day to help him know when it was bed time and what time it was ok to get out of bed in the mornings (the little monkey had a horrible habit at waking up at 5am on a Saturday morning!). I originally coded up a simple Python Script to control the lights from scratch but then [Tanya](https://twitter.com/tanurai) at Pimoroni wrote up a [really nice guide](https://learn.pimoroni.com/tutorial/tanya/cute-alarm-clock) using the Schedule library that I adapted for my own needs.

This is a really quick and easy way to build a nightlight. You just grab a Raspberry Pi Zero W, slap on a Unicorn Hat, pop in a little SD card and throw up some simple Python code and you'll be up and running. Very little electronics knowledge is needed because everything is practically plug and play! All thats left to figure out is an enclosure. I find the Tiger/Flying Tiger chain of stores to be a great source for fun, hackable light enclosures (however, if you have a 3D printer, the sky really is the limit!)

The downside is cost. A Raspberry Pi Zero WH will set you back around £15, The Unicorn Phat is £10 and you're looking at about another £7 for an SD card (depending on size). Finallyt for power, the The official Pi Foundation power supply is £8. This comes to a total of about £40. It would be cheaper to use a standard Pi Zero but unfortunately the Pi doesn't have a real time clock so you're dependent on an internet connection in order to get the current time using NTP (Network Time Protocol).

Scaling up and reducing costs
==

In recent years I've had a couple more kids and more kids means more night lights. So I wanted to find a cost-effective way to scale up production so I could make lights for all the kids without breaking the bank.

So here's the plan: I have a Pi 3B+ in my office that I use as a server (it runs a wireless print server, ad blockers and controls some lights in my office). I will use that to gather the time and beam it to the lamps wirelessly. The lamps themselves will be controlled with the amazingly cheap ESP8266 (Arduino IDE compatible Wifi enabled microcontroller!) which can be ordered direct from AliExpress for less than £2 per unit.

For lights, I bought WS2812B RGB LED strips (also from AliExpress), a 1 meter strip costs as little as £3 and includes 60 LED's. 15 LED's is enough to supply more than enough light for our needs so that's 4 lamps from one strip! For power, I use a standard USB phone charger and micro USB cable (I had these lying around but you can buy the charger and cable for about £2 depending on the length of the cable). So all in, you can build each lamp for less than £5!

I can already hear people shouting:

> **"WHY NOT JUST GET THE TIME DIRECTLY ON THE ESP8266 AND SKIP THE PI ALTOGETHER???"**

<center><iframe src="https://giphy.com/embed/p2WRqA5wmXQuA" width="480" height="328" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></center>
    
You could absolutely use a variety of methods to get the time from the internet on the ESP8266 and use that as the foundation of your logic. However, if you want to update the timing, you have to disassemble the lamp, connect the ESP8266 to your computer, re-upload the code and repeat for every single lamp you own. That's more hassle than I have time for. By putting the majority of the logic on the Pi, I can easily update the code from anywhere in my home, over the network and without removing the lights from their various random locations.

NB - you don't even have to put the logic on a Pi - if you have a web server anywhere online that supports scripting, you could host your backend code there. I used Python, but you could absolutely do this with PHP, Ruby, heck even Perl if you're that way inclined.

Building the lights:
==

<img src="/static/img/IMG_7437.jpg">

Time for some soldering! Trim the LED strips into appropriate lengths, cutting along the cut marks and solder on some stiff hookup wire making sure to attatch them to the data-in side of the strip. Attatch the positive wire to the 5v pin on the ESP8266, the ground to ground and the data pin to D3 (you can change this but you'll also need to update the code). I then gently loop over the strip and hotglue it to itself to create a "bulb" of sorts. Rinse and repeat for each lamp. I also add a little hot glue around the wires for strain relief. Upload the code and stick it in the enclosure of your choosing.

The Raspberry Pi
==

If you haven't done this already, get a Raspberry Pi up and running with the latest Raspian and [install Flask using PIP](https://projects.raspberrypi.org/en/projects/python-web-server-with-flask/2). Make sure it's connected to your internet.

The code here is pretty simple, it fires up a simple web server using Flask, gets the time from the system and then converts the time into a decimal for easy handling (for example 07:30 becomes 7.5). It then uses some simple logic to define the colours and brightness and then concatenates them into a simple string that it outputs as a very basic text document.

In your home directory, create a file called `app.py` and add the following code:

{% highlight python %}

from flask import Flask,request
import datetime
import random

app = Flask(__name__)

@app.route('/')

def index():

	user = request.args.get("user")

	# get the time
	a = datetime.datetime.now().time()

	# convert time to a float
	mytime = a.hour+a.minute/60.0

	# default to off
	color = "0,0,0,0"

	if(mytime > 7):

		r = 0
		g = 255
		b = 0
		a = 255

	if(mytime > 9):

		r = 255
		g = 255
		b = 255
		a = 255

	color = str(r) + "," + str(g) + "," + str(b) + "," + str(a)

	return color

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=999)

{% endhighlight %}

This is a simple example that starts off switched off, turns green at 7am and then white at 9am. You can add additional logic if you want more steps in your lights (in mine I have them start off red at 5am and slowly fade through yellow to white to simulate a sunrise then I have additional logic that sets them to my kids favourite colours during the day).

You can fire up the web server by simply typing `sudo python app.py` on the command line and then you can visit the server in your browser using it's hostname or IP to see if it's working. Assuming the name of your Pi on the network is "mypi" you should be able to see your Pi at `http://mypi.local:999` and the output in your browser should be something like `255,0,0,255`.

You can now disable the web server using `ctrl+c`

Next we need it to boot up automatically when the Pi is switched on. That's easy, just type `crontab -e` on the command line and then scroll to the bottom and add the following line: 

`@reboot /home/pi/app.py`

Then just to be sure, reboot the Pi (`sudo reboot`) and then test the URL in the browser again. If all is well, you can move onto putting the software onto the ESP8266!

The ESP8266
==

<img src="/static/img/IMG_7435.jpg">

What happens on the ESP8266 is pretty simple. It connects to the network then performs a http get request to a web page where it expects to see four values separated by commas in plain text; a red, green and blue value to generate an RGB colour and a brightness value. It chops this string into individual numbers and then pushes them out to the LED strip using the FastLED library. It'll repeat this every 60 seconds or so (this is adjustable depending on how close to real time you need your lights to be).

Simply copy this code into the Arduino IDE and modify the constant definitions at the top of the file to suit your needs. You'll need to add your wifi details and the address to the web server that's serving up the colours and you'll need to install the FastLED and ESP8266 Libraries in the Arduino IDE. Once you're happy compile and publish the code to the ESP8266

{% highlight python %}

#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <FastLED.h>

FASTLED_USING_NAMESPACE

#define DATA_PIN    3
#define LED_TYPE    WS2812
#define COLOR_ORDER GRB
#define NUM_LEDS    16
#define WIFI_SSID   "your wifi network name here!"
#define WIFI_PWD    "Your wifi password"
#define HOST_NAME   "NightLight01"
#define DATA_SRC    "URL for the location of your little API"


#define BRIGHTNESS          90
#define FRAMES_PER_SECOND  120

CRGB leds[NUM_LEDS];

// nasty function to explode strings like in PHP
String getValue(String data, char separator, int index) {
  int found = 0;
  int strIndex[] = {0, -1};
  int maxIndex = data.length()-1;
  for(int i=0; i<=maxIndex && found<=index; i++){
    if(data.charAt(i)==separator || i==maxIndex){
        found++;
        strIndex[0] = strIndex[1]+1;
        strIndex[1] = (i == maxIndex) ? i+1 : i;
    }
  }
  return found>index ? data.substring(strIndex[0], strIndex[1]) : "";
}

void setup() {

  Serial.begin(230400);

  // tell FastLED about the LED strip configuration
  FastLED.addLeds<LED_TYPE,DATA_PIN,COLOR_ORDER>(leds, NUM_LEDS).setCorrection(TypicalLEDStrip);

  // set master brightness control
  FastLED.setBrightness(BRIGHTNESS);

  WiFi.begin(WIFI_SSID, WIFI_PWD);
  WiFi.hostname(HOST_NAME);

  while (WiFi.status() != WL_CONNECTED) {
    delay(100); 
    Serial.println("Connecting.");
  }

}

void loop() {

  delay(1000);

  if (WiFi.status() == WL_CONNECTED) { //Check WiFi connection status

    HTTPClient http;                              //Declare an object of class HTTPClient
    http.begin(DATA_SRC);  //Specify request destination
    int httpCode = http.GET();                    //Send the request

    if (httpCode > 0) { 

      String payload = http.getString();   
      Serial.println(payload);             

      int r = getValue(payload, ',', 0).toInt();
      int g = getValue(payload, ',', 1).toInt();
      int b = getValue(payload, ',', 2).toInt();
      int a = getValue(payload, ',', 3).toInt();

      FastLED.setBrightness(a);

      for (int i=0; i <= NUM_LEDS; i++) {
        leds[i].setRGB(r,g,b); 
      }
      FastLED.show();

    }

    http.end();   //Close connection

  }

  delay(1000*60);    //Send a request every 60 seconds

}

{% endhighlight %}

One great tip is to add a parameter to the data_src URL, then you can use it to identify individual lights in your Python code and respond with different colours depending on the device requesting it (for example http://mypi.local/?user=jack). Just don't forgot to change the parameter for each light you're creating (my python code above is already reading the user value from the query string, you can access it with the `user` variable anywhere in the Python script)

If all went to plan, you should now have a networked night light up and running that changes colour depending on the time of day. Of course, there's no need to limit yourself. Your Python script can decide it's colours in all kinds of ways, you could randomly generate colours, set them based on the weather or even pull data from a public API like Cheerlights and let Twitter decide the colour of your lights! There's also no need to wrap them in a kids nightlight - you could install the lights in plant pots, on headboards or even behind the TV. Once you have the basic pieces described here, the rest is all up to your imagination.

Thanks for taking the time to read this. I would love to see what you made, if you do make this, please share your projects, suggestions and questions with me on Twitter ([@awarburton](http://twitter.com/awarburton)).
