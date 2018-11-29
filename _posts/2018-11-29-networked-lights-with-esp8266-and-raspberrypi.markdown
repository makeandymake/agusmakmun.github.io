---
layout: post
title:  "Networked Nightlights with ESP8266 and Raspberry Pi"
date:   2018-11-29 21:00:00 +0100
---
A couple of years ago I made my eldest kid a nightlight for his bedroom. It was a simple project using a Raspberry Pi Zero, a Pimoroni Unicorn HAT and some code in Python. It was especially useful because we were able to use the colours to help him know when it was an appropriate time to get out of bed in the morning (the little monkey had a horrible habit at waking up at 5am on a Saturday morning and waking the whole house up!). I originally coded up some horrible Python but then [Tanya](https://twitter.com/tanurai) at Pimoroni wrote up a [really nice guide using](https://learn.pimoroni.com/tutorial/tanya/cute-alarm-clock) the Schedule library that I adapted for my own needs.

The good thing about this way of doing it, is you can build it really quickly. Take a Raspberry Pi Zero W, slap on a Unicorn Hat, pop in a little SD card and throw in some simple Python code and you're up and running. Very little electronics knowledge needed because everything is practically plug and play! All thats left to figure out is an enclosure (the Flying Tiger chain of stores make some awesome plastic battery powered night lights that are great fun to hack and come in relatively cheaply)

The downside is cost. A Raspberry Pi Zero WH will set you back around £15, The Unicorn Phat is £10 and you're looking at about another £7 for an SD card (depending on size). That's £32 and thats not including a power supply or enclosure. So why not use the standard, much cheaper Pi Zero? Well unfortunately the Pi doesn't have a real time clock so you need an internet connection so you can figure out the current time using NTP (Network Time Protocol).

Scaling up and reducing costs
==

Since I built the first light, I've had a couple more kids and more kids means more nightlights. So I wanted to find a more cost-effective way to scale up my production of nightlights so I could make one for sprog numbers two and three as well (plus reclaim the hardware from the original lamp to use in other projects!). How can I make 3 lamps for less?

So here's the plan - I have a Pi 3B+ in my office that I use as a server. I will use that to gather the time and beam it to the lamps wirelessly. The lamps themselves will be controlled with the amazingly cheap ESP8266 (Arduino IDE compatible Wifi enabled microcontroller!), which can be ordered direct from China for less than £2 per unit. For lights, I bought WS2812B RGB LED strips from Aliexpress, a 1m strip costs as little as £3 and includes 60 LED's. 15 LED's is enough to supply more than enough light for our needs so that's 4 lamps from one strip! For power, I use a standard USB phone charger and micro USB cable (I had these lying around but you can buy the charger and cable for about £2 depending on the length of the cable). So all in, you can build each lamp for about £5!

Now before I go any further, I can hear people already shouting: 

"WHY NOT JUST GET THE TIME DIRECTLY ON THE ESP8266 AND SKIP THE PI ALTOGETHER???"

That is a perfectly valid question. You could absolutely use a variety of methods to get the time from the internet on the ESP8266 and use that as the foundation of your logic. But, this means that if you want to update the logic, you have to disassemble the lamp, connect the ESP8266 to your computer, reupload the code and repeat for every single lamp you own. That's more hassle than I have time for. By putting the majority of the logic on the Pi, I can easily update the code from anywhere in my home over the network without removing the lights from their various random locations.

NB - you don't even have to put the logic on the Pi - if you have a web server anywhere that supports scripting, you could host your backend code there. I used python, but you could absolutely do this with PHP, Ruby, heck even Perl if you're that way inclined.

Building the lights:
==

Time for a little bit of soldering. Trim the LED strips into appropriate lengths, cutting along the cut marks and solder on some stiff hookup wire making sure to attatch them to the data-in side of the strip. I use red for positive, black for ground and a nice bright colour for data. Attatch the positive wire to the 5v pin on the ESP8266, the ground to ground and the data pin to D3 (you can change this but you'll also need to update the code). I then gently loop over the strip and hotglue it to itself to create a "bulb" of sorts. Rinse and repeat for each lamp. I also add a little hot glue around the wires for strain relief. Upload the code and stick it in the enclosure of your choosing.

Code on the ESP8266
==

So I'm going to assume that you know how to operate the Arduino IDE and upload code to your ESP8266. Simply copy this code into the Arduino IDE and modify the constant definitions at the top of the file to suit your needs. You'll

{% highlight ruby %}

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
    // flash on boot
    for (int i=0; i <= NUM_LEDS; i++) {
      leds[i].setRGB(255,0,0);
      FastLED.show(); 
    }
    delay(100);
    // flash on boot
    for (int i=0; i <= NUM_LEDS; i++) {
      leds[i].setRGB(0,255,0);
      FastLED.show(); 
    }
    delay(100);
    // flash on boot
    for (int i=0; i <= NUM_LEDS; i++) {
      leds[i].setRGB(0,0,255);
      FastLED.show(); 
    }
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

