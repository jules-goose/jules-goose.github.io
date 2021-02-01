---
layout: post
title: Making an esp8266 powered led strip, part 1
---

recently i've been working on a custom led strip controller, principaly to for a low-cost light bar project but that could be reused in other project. 
requirement were to have a mean to connect the strip to new wifi networks (which mean a connection portal type of deal), to have the ability to control the strip through a locally hosted web dashboard,to subscribe to mqtt brokers, record actions and bind them to mqtt payloads, and the ability to store all of these information in a local file system so these information remain, finally, as far as the framework choice go, i chose to use the esp8266 arduino core, as i am much more familiar with it and most library i used are originally meant for the arduino family of mcu and since another motivation was to learn how to use platformio
![ahah led go shiny]({{ site.baseurl }}/images/blogpost-1/led-strip-blogpost-p1.png "illustration")

## a wifi connection portal

my first move was to write a wifi connection portal, which surprisingly enough, doesn't really already exist(at least as far as i know), for this purpose, i've made use of the esp8266 arduino core filesystem library LittleFS, which allow for an use of some of the flash chip as a file system, with folder and so on

```
|--------------|-------|---------------|--|--|--|--|--|
^              ^       ^               ^     ^
Sketch    OTA update   File system   EEPROM  WiFi config (SDK)
```

so the way this work is, on boot/power on, during the setup phase, the mcu will read through a file that contain registered / previously configured access point and attempt to connect to them,if it succeed to connect to an ap, it will then go on and setup the web dashboard(which will be discussed later on), if it fail, and once all access point have been itterated through and all the connection attempt have timed out, it will then scan the locally available wifi access point and start a webserver (using ESPAsyncWebServer) as well as an access point meant for the user to connect to,the webserver only serve a webpage with a form which contain a dropdown list of local access point and a password input field, once you submit that form, the selected accesspoint and inputted password get put in the credential list and the device then need to be restarted, once it is restarted, it'll rince and repeat the previous connection step.

for more information and to take a look at the proof of concept module, see : [WebConnectionPortal repository](https://github.com/jules-goose/webConnectionPortal)

## a locally hosted dashboard 

once the wifi connection portal was done, the next step was to write a web control dashboard (still a work in progress, since it's currently only a proof of concept and isn't aesthetically pleasing)
so i first had to write a webserver that allowed me to reuse a part of my web component, firstly because i wanted to use as little of a file system as i could, and secondly because it meant less work in the future, so i made use of the ESPAsyncWebServer library and came up with the following piece of code ,which all happen during the setup phase (for simplicity sake, i'm not including the wifi connection portal step)

```cpp
void setup()
{
    
    server = new AsyncWebServer(80);
    server->on("/", HTTP_GET, [](AsyncWebServerRequest *request)
    {
      handleResponse("dashbrd",request);
    });
    server->on("/actionconfig", HTTP_GET, [](AsyncWebServerRequest *request)
    {
      handleResponse("actionconf",request);
    });
    server->on("/ledstripconfig", HTTP_GET, [](AsyncWebServerRequest *request)
    {
      handleResponse("stripconf",request);
    });
    server->on("/brokerconfig", HTTP_GET, [](AsyncWebServerRequest *request)
    {
      handleResponse("mqttconf",request);
    });
}
```
and you may be asking yourself , what is handleResponse ? it's not a ESPAsyncWebServer method, and you're right, it's a method i wrote to build a response webpage, i don't really know where it falls , it's not really Server side rendering since i do not render the page / do not have a processor function that would take value and put them at tag on the webpage (although it is possible using ESPAsyncWebServer). anyway, here is the method, it take a contentID , which is used to recover files with the web page content, this way i can use the same header and footer element (which contain link to front end libraries i use as well as custom style and the navbar which is common to every view)
```cpp
void handleResponse(String contentId,AsyncWebServerRequest *request)
{
  AsyncResponseStream *response = request->beginResponseStream("text/html");
  response->addHeader("Server","ESP Async Web Server");
  File header = LittleFS.open("/www/led/header.html","r");
  response->print(header.readString());
  header.close();
  String contentPath;
  contentPath= "/www/led/"+contentId+".html";
  File content = LittleFS.open(contentPath,"r");
  response->print(content.readString());
  content.close();
  File footer = LittleFS.open("/www/led/footer.html","r");
  response->print(footer.readString());
  footer.close();
  request->send(response);
}
```
this method/way of doing allow me to reuse a lot of content, which in turn save some space and reduce the work load 

## controlling the led strip

for now i'm only using one led (taken from the led strip i ordered) for testing purposes as i have yet to receive the rest of the parts needed to drive the rest of the strip safely (you have to understand, i'm not a fan of setting my house on fire), so for now, it's only 1 led to mostly test functionalities and so on, to control the led strip, i'm using a form that takes in a led strip length as a well as a color value (which is also set to the current led color when the dashboard is loaded)

see, i've learnt that both FastLed and NeoPixel both can't set up the length of a led strip at runtime, from what i've understood, it has to do with code optimisation and memory management, so the solution i've found is to set the strip length value to something like 200 and then use the form length value as my index instead of the one used to define the CRGB object(see example bellow)

```cpp

#define LED_PIN    5
#define NUM_LEDS 200

CRGB leds[NUM_LEDS];//create a crgb object with 200 separate leds
AsyncWebServer *server;

long ledlength = 20;
int ledcolor = 0x0000FF
void setup()
{
    
    FastLED.addLeds<NEOPIXEL, LED_PIN>(leds, NUM_LEDS);
}
void loop()
{
    for(int idx=0;idx<ledlength;idx++)
    {
      leds[idx] = ledcolor;  
    }
    FastLED.show();
    ledlength = randNumber(0,20);
    delay(250);
}
```
as you can see, the led strip object ,program wise, is 200 leds long, but because we use ledlength as our max value, we can control a 20 led long strip, if the code was made in such a way as to accept a led strip length value , you could drive any length of ledstrip as long as it's not longer than 200

## now what ? 
well, that's it for part 1, in part 2 i will cover the creation of "action", like, associating a color,intensity,pattern with a mqtt payload, as well as connecting to a mqtt broker and configuring said connection using the web page interface, then in part 3, i'll cover the hardware side of thing(which will come out when i receive the pcb and all the parts)