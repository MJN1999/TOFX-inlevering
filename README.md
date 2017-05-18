
[PDF version avalible here](https://www.mediafire.com/?77oiuq5e55ifuoc)

<!-- toc orderedList:0 depthFrom:1 depthTo:6 -->

* [The Arduino and ESP8266 code](#the-arduino-and-esp8266-code)
    * [Setup the Arduino IDE](#setup-the-arduino-ide)
    * [Software serial (library)](#software-serial-library)
    * [Arduino JSON (Library)](#arduino-json-library)
    * [Next we'll define some variables we'll use later.](#next-well-define-some-variables-well-use-later)
    * [The setup function](#the-setup-function)
    * [Setting up the ESP8266 to fit our needs](#setting-up-the-esp8266-to-fit-our-needs)
    * [The loop() and reciever logic.](#the-loop-and-reciever-logic)
    * [This is then our final piece of code.](#this-is-then-our-final-piece-of-code)
* [Testing the connection](#testing-the-connection)
    * [Step 1: debugging the arduino.](#step-1-debugging-the-arduino)
    * [Step 2: Connect to the TCP server.](#step-2-connect-to-the-tcp-server)
* [Android / TCP client](#android-tcp-client)
  * [Intro](#intro)
  * [Make an android project](#make-an-android-project)
  * [Making the UI](#making-the-ui)
  * [The code](#the-code)
    * [onCreate](#oncreate)
  * [Enabling Developer options and android debugging](#enabling-developer-options-and-android-debugging)
    * [Making it all work (Connecting to the Arduino from the Phone)](#making-it-all-work-connecting-to-the-arduino-from-the-phone)
    * [Implimenting the TCP client finally](#implimenting-the-tcp-client-finally)
    * [Finnishing up the android app](#finnishing-up-the-android-app)
    * [Final android](#final-android)
  * [You can also download the android code I wrote if the android part of this handin was hard to follow.](#you-can-also-download-the-android-code-i-wrotehttpswwwmediafirecomvkqh6bdgegd4kgo-if-the-android-part-of-this-handin-was-hard-to-follow)

<!-- tocstop -->


# The Arduino and ESP8266 code
This page will cover the code for the arudino to make it work with the ESP8266 chip aswell as cover the code and setup for the ESP8266 chip. I highly recommend doing some basic arudino coding and examples and projects found in the book before trying to make sense of all of this code. Links to relevant hand helpfull resources will be given troughout this page aswel.

### Setup the Arduino IDE
First of all we need to install the Arduino IDE you can find its download page [here](https://www.arduino.cc/en/Main/Software), and then install it. After installation open up the Arduino IDE and youll be precented with a new scetch.

### Software serial (library)
We'll start off by including the free library [SoftwareSerial](https://www.arduino.cc/en/Reference/softwareSerial), the library allows us to have more then one serial port by allowing the usage of data pins, this is usefull as the Arudino UNO has only one Serial conneciton built in that being the USB port.

Include the library by selecting from the menu `Sketch->Include library->SoftwareSerial` it should append the following to the begging of our sketch.

```c
#include <SoftwareSerial.h>
```

### Arduino JSON (Library)
The next dependancy we will need is [ArduinoJSON](https://github.com/bblanchon/ArduinoJson) the library is a arduino compatible implimentation of [JSON](https://no.wikipedia.org/wiki/JSON) which is a really handy string formating standard. We'll install it by selecting from menu `Sketch->Include Library->Manage Libraries` type in JSON and install `ArduinoJson` by Benoit Blanchon (It should be the first result) then close the Manage Library window and now inport the lib by going to `Sketch->Include Library->ArduinoJson` and it should append
```c
#include <ArduinoJson.h>
```

---

### Next we'll define some variables we'll use later.

---
For self redo this with "Well need to define a ..... then put the code"

---


`SoftwareSerial ESP8266 (3,2);` Makes a software serial, the RX pin is 3 and the TX pin is 2, these are data pins and can be mappen to any datapin by changing the argument.

`StaticJsonBuffer<200> jsonBuffer;` Makes a Json bufferd well use to parse the json later, please note if you for some reason need to parse json thats bigger then 200 char long change the value to the requierd amount of chars.

`String Ssid = "Båt #1";` This is the SSID (Wifi name) that will showup when we look for and connect to our WIFI.

`String Pass = "awesomepass42"`This is the password for our WIFI network

`String Channel = "5";` This is the [channel the WIFI network](https://www.wikiwand.com/en/List_of_WLAN_channels) will be on.

The last variable for our WIFI is the auth protocol we'll use for the WIFI, You can find a list of them here, but I recommend using WPA2_PSK as its the industry standard and supported by basically everything. `String Encoding = "3"`

We define the port for the [TELNET](https://www.wikiwand.com/en/Telnet) server like so `String Port = "1336";`

For debugging purposes we'll setup some debug variables, the debug serial is the refresh rate of the serial, the default for USB debugging is 9600.
```
boolean debug = true;
int debugserial = 9600;
```
---
### The setup function

[The Arduino language](https://www.arduino.cc/en/Reference/HomePage) has some nifty constant functions/voids we'll take use of spesifically `void setup()` [which is ran at bootup](https://www.arduino.cc/en/Reference/Setup), and `void loop()`[which runs peridocially while the arduino is powerd on](https://www.arduino.cc/en/Reference/Loop).

In our `void setup()` we'll setup the debug serial so we can use it for debugging, and also call a function we'll make later. If you have problems understanding the code id recommend you look trough the language refrence for [Serial](https://www.arduino.cc/en/Reference/Serial) (however in short we are just starting a serial at the frequency spesified in our debugserial variable we defined earlier) and [IF statment](https://www.arduino.cc/en/Reference/If) aswel as [the void datatype](https://www.arduino.cc/en/Reference/Void)
```c
void setup() {
  if(debug)
  Serial.begin(debugserial);

  setupESP8266();
}
```
---
### Setting up the ESP8266 to fit our needs

The ESP8266 is an highly adaptive and featurerich chip, however it does need some setting up we will set it up using AT commands over the SoftwareSerial ESP8266 we defined earlier. Please refer to [this page](https://www.google.no/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwjh3O6Wx_HTAhXCF5oKHVd0ASUQygQIOTAC&url=https%3A%2F%2Fwww.itead.cc%2Fwiki%2FESP8266_Serial_WIFI_Module%23AT_Commands&usg=AFQjCNFeavYqiofUXwon0x-9UzvyxRYi-Q&sig2=9z5VRupZOVDFmjPY1vz4yA) for a list of AT commands and a more indepth explanation of them.

First we make a send function.

```c
void send(String data, int sleep){  
  ESP8266.println(data);
  delay(sleep);
}
```
`ESP8266.println(data)` sends the data to the ESP8266 over the ESP8266 serial we have defined earlier. However the unit may take some time to complete the setup instructions, to make sure the unit completes all instructions and does not skip any by getting overwvelmed we make use of the [delay(time)](https://www.arduino.cc/en/Reference/Delay) function.

Then we will define the ESP8266 function we called ealier in our `void setup()`. There is quite a lot happening here so I recommend reading my walktrough slowly and to have the [AT command refrence sheet](https://www.itead.cc/wiki/ESP8266_Serial_WIFI_Module#AT_Commands) handy.
```c
void setupESP8266(){  
  ESP8266.begin(115200);

  send("AT+RST", 5000);
  send("AT+CWMODE=2",2000);
  send("AT+CWSAP="" + Ssid + "","" + Pass +"","+Channel+","+Encoding,2000);
  send("AT+CIPMUX=1", 1000);
  send("AT+CIFSR",1000);
  send("AT+CIPSERVER=1,"+Port,1000);
}
```
First we startup our ESP8266 serial with  `ÈSP8266.begin(115200);`this means the serial is now live and has a "serial rate" of 115200 (I could not find any information about how fast a "serial rate" is however the higher the number the faster.

Anyhow we'll first reset the ESP8266 unit to make sure we are working with a clean chip. Using the `send`method we defined earlier we will call the AT+RST command and wait 5s for it to complete its reset.
`send("AT+RST", 5000);`

We'll then set the chip in AP mode so it hosts its own WIFI for us to connect to, an less convinent alternative is to put it in Sta mode which will make it connect to an exisit WIFI network in the building we are in. However since those are not always avalible i strongly recommend AP mode using [the refrense sheet](https://www.itead.cc/wiki/ESP8266_Serial_WIFI_Module#AT_Commands) you strongly should have open by now we see that we need to use `AT+CWMODE`with a value of to so we send that to the ESP8266 unit with `send("AT+CWMODE=2",2000);`

Next we tell the Unit using the `AT+CWSAP` command how to setup its WIFI. The `AT+CWSAP`command requiers 4 arguments:

 1. SSID
 2. Password
 3. Channel
 4. Auth/encoding system

We pass it our variables we defined at the start of our code and send it with `send("AT+CWSAP="" + Ssid + "","" + Pass +"","+Channel+","+Encoding,2000); `

Now since the server we will setup in a little bit takes up a connection and the ESP8266 by default only allows one connection at time we will need to tell it to allow multiple connections, we will use the `AT+CIPMUX` AT command for this `send("AT+CIPMUX=1", 1000);`

We'll tell the ESP8266 to get an IP address for itself `  send("AT+CIFSR",1000); ` this now completes the WIFI setup and then we start the TCP telnet server using `send("AT+CIPSERVER=1,"+Port,1000);`

---
### The loop() and reciever logic.
As we touched on earlier another curcial constant void for us is the `void loop()`that executes code periodically as the Arduino is running. We will use this to handle our reciver logic.

First we get the input from the ESP8266 with the following code:
```
 while (ESP8266.available()){
 String inData = ESP8266.readStringUntil('\n');
```

`while (ESP8266.avalible())` checks if the ESP8266 unit is avalible and ready to send data at this spesific time.
if it is avalible we read a full line of it and store it in a String we call inData with ` String inData = ESP8266.readStringUntil('\n');`.

The arduino and ESP8266 is now finished the indata is what the auruino will recive from our client. For now we can see the indata with the following debug code to print out the data we recive into our USB debug serial.
```c
     if(debug)
     Serial.println(Ssid+ "'s ESP8266: " + inData);
```

---



### This is then our final piece of code.
```c
#include <ArduinoJson.h>

#include <SoftwareSerial.h>

SoftwareSerial ESP8266 (3,2);
StaticJsonBuffer<200> jsonBuffer;
String Ssid = "båt #1";
String Pass = "awesomepass42";
String Channel = "5";
String Encoding = "3";
String Port = "1336";
boolean debug = true;
int debugserial = 9600;

void setup() {
  // put your setup code here, to run once:
  if(debug)
  Serial.begin(debugserial);
  setupESP8266();
}

//Startup code
void setupESP8266(){
  ESP8266.begin(115200);
  send("AT+RST", 5000);
  send("AT+CWMODE=2",2000);
  send("AT+CWSAP="" + Ssid + "","" + Pass +"","+Channel+","+Encoding,2000);
  send("AT+CIPMUX=1", 1000);
  send("AT+CIFSR",1000);
  send("AT+CIPSERVER=1,"+Port,1000);
}

void send(String data, int sleep){
  ESP8266.println(data);
  delay(sleep);
}

//Our constnatly running code
void loop() {
     while (ESP8266.available()){
     String inData = ESP8266.readStringUntil('\n');

     if(debug)
     Serial.println(Ssid+ "'s ESP8266: " + inData);
  }
}
```

# Testing the connection
Before we proceed making an Android app, we should check if the TCP connect works.

### Step 1: debugging the arduino.
Connect (if not already connected) the arduino the the PC open the arduino IDE, make sure the arduino is connected under `tools->Port->Com(X) (Arduino/Genuino Uno)` then click the magnifying glass icon in the top right corner of the IDE, this opens the Serial monitor. make sure the bandrate is set to 9600 and no line ending is selected. Then upload the sketch we made earlier to the arduino (by clicking the upload icon). wait for about 10-20 sec and some text should appear in the serial monitor. if the text is il-legiable make sure all the cabels are properly connected, and try again.

### Step 2: Connect to the TCP server.
Download some tcp client I used [Mobile telnet](https://play.google.com/store/apps/details?id=mobiletelnet.feng.gao&hl=no) for android. I then turned off mobile data and connected to the WIFI of the arudino. Open up the Telnet app click the topright and navigate to Telnet-Settings. The default ip for the arduino is `192.168.4.1` and the port is the one specified in the code in this case its `1336` enter these values and click OK. Now click Connect and in the serial monitor we should see "CONNECT" or some variation of it. If we now type a message in the telnetapp and send it, we should see it showup in the Serial monitor.

---


# Android / TCP client

## Intro
Now that we got our TCP server setup, we need a client to connect to it. We can do this trough any tcp client but we are gonna do this a fancy way and make an android app to do so.

Now i highly recommend looking at [the official android tutorials](https://developer.android.com/training/basics/firstapp/index.html) for a more indepth look at how an android app works aswell as how to write one. As android is very extensive i cant cover all of it in this handin, the following is a quick start guide.

First of all [Download](https://developer.android.com/studio/index.html) and install Android Studio.

## Make an android project
Open android studio and click after setting up your settings by clicking `Start a new Android Studio Project`.
Change the applicaiton name and domain as you self see fit, I Will use "Arudino TCP client" for the name, and "Marius.TOFX" for the domainame, no c++ support is needed tough you can include it if you want.

Select the devices you want the app to run on, The app im making is only for phone/tablet and has a Minimum SDK of android 5.0 lolipop, and we want an Empty activity. keep the acitvity name as it is default. It will take a little while for our dev enviroment to setup fully we can spend this time thinking of what design we want the sliders to be. when the bar at the bottom of android studio is done and its no longer saying "indexing" the project is ready.

## Making the UI

we should now be in a file called activity_main.xml and see a designer for the UI. Start off by delting the "hello world", by selecting it -> rightclick delete, we'll now make our UI, my UI will have two sliders and one button.

If you wanna replicate my ui follow these steps

1. Drag in a `RelativeLayout`
2. Place a button in the center.
3. Place a Seekbar in the Top and Resize it fill the horizontal space.
4. Place another Seekbar but this time in the Bottom and resize it to fill the horizontal space.
5. Open the UI XMl code by clicking "Text" in the Desing view.
6. Under button add `android:rotation="90"` this will rotate our button 90 degrees.
7. Under RealtiveLayout change Layout_Width and Layout_Height to = "match_parent".
8. Change the ID of the button and seekbar to "Con","Ror","Mot"
this is then my final code for the UI.
```XMl
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="tofx.marius.arduinotcpclient.MainActivity">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:layout_editor_absoluteX="8dp"
        tools:layout_editor_absoluteY="8dp">

        <Button
            android:id="@+id/Con"
            android:rotation="90"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerHorizontal="true"
            android:layout_centerVertical="true"
            android:text="Button" />

        <SeekBar
            android:id="@+id/Ror"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentStart="true"
            android:layout_alignParentEnd="true" />

        <SeekBar
            android:id="@+id/Mot"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentBottom="true"
            android:layout_alignParentStart="true"
            android:layout_alignParentEnd="true" />
    </RelativeLayout>
</android.support.constraint.ConstraintLayout>
```

## The code

I highly recommend having the [Android refrense sheet](https://developer.android.com/reference/packages.html) open, I also recommend if you are confused by the android code to look at some of the [Android tutorials](https://developer.android.com/training/index.html) to help you wrap your head around Android and java in general.

Click `Project` on the right of android studio then under java open the first object and open `MainAcivity` it should look somewhat like this.

```java
package tofx.marius.arduinotcpclient;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.Window;
import android.view.WindowManager;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

### onCreate
oncreate is basically Androids version of `int main()` or arduions `void setup()`
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
}
```
First since we want this app to be a fullscreen app we'll insert this peice of code into it.

```java
ActionBar actionBar = getSupportActionBar();
  actionBar.hide();
  getWindow().getDecorView().setSystemUiVisibility(
          View.SYSTEM_UI_FLAG_LAYOUT_STABLE
        | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
        | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
        | View.SYSTEM_UI_FLAG_FULLSCREEN
        | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);
```
`ActionBar actionBar = getSupportActionBar();` Gets the actionbar, and `actionBar.hide();` hides it.
```
getWindow().getDecorView().setSystemUiVisibility(
          View.SYSTEM_UI_FLAG_LAYOUT_STABLE
        | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
        | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
        | View.SYSTEM_UI_FLAG_FULLSCREEN
        | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);
```
Makes our app fullscreen


A note for the Android Studio is that if we need to import anything it will be highlighted red, hover over the text and select import, this will automatically import whats needed which in our case is the View class.

```java
import android.view.View;
```


Anyhow our `onCreate()` now looks like this
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    ActionBar actionBar = getSupportActionBar();
    actionBar.hide();
    getWindow().getDecorView().setSystemUiVisibility(
            View.SYSTEM_UI_FLAG_LAYOUT_STABLE
              | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
              | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
              | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
              | View.SYSTEM_UI_FLAG_FULLSCREEN
              | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);
}
```

## Enabling Developer options and android debugging
On your android phone open up your settings app, find "About phone"/"About device" or some other verious of that. in `About Phone` find tap the `Build number a few times` eventually a message saying "developer options was enabled"/"You are now a developer" should pop up. If not please google your phones name along witht he search query "Android Enable Developer options".

After enabling Developer Options you should now see `Developer options` in the settings menu in android, make sure its marked as "on" then find and enable
* Android debugging
* ADB over network (Optional)
* ADB notification (Optional but highly recommended)

Now connect your phone(while its unlocked/ at the home screen) by USB to your computer, A popup asking for permission to connect is most likly gonna popup click remember fingerprint and allow it to debug. in your ntoficiation drawing there should be a notice android debugging over usb is enabled. next go to android studio and click the "Run" button (Green arrow) select your phone.

After the app is installed you can now disconnect your phone from the computer and try out the app on your phone, repeate these steps of designing the UI and debugging  till the UI works and looks as you want. note: you may need to uninstall the app from your phone before you can load a new dev version by clicking run agian. For refrence my UI has two sliders and 1 button.

### Making it all work (Connecting to the Arduino from the Phone)
Now that our UI looks and behaves as we want it to we'll add the part of the code that will connect to our Arduino.

First deifne these objects
```java
   SeekBar MotBar;
   SeekBar RorBar;
   Button  ConBtn;
   Boolean Connected = false;
   Socket socket;
   OutputStream socketstream;
   PrintWriter output;



```
Then well initislize these in our `onCreate` by grabbing them from our UI add the following code to onCreate
```java
     MotBar = (SeekBar)findViewById(R.id.Mot);
     RorBar = (SeekBar)findViewById(R.id.Ror);
     ConBtn = (Button)findViewById(R.id.Con);
```     

We'll now make the requierd Listeners which we'll fill in later.

make the Seekbar listener
```java
SeekBar.OnSeekBarChangeListener ChangeValue = new SeekBar.OnSeekBarChangeListener() {
       @Override
       public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {

       }

       @Override
       public void onStartTrackingTouch(SeekBar seekBar) {

       }

       @Override
       public void onStopTrackingTouch(SeekBar seekBar) {

       }
   };
```
We'll subscribe our seekbars to it by inserting the following code in `onCreate`
```java
MotBar.setOnSeekBarChangeListener(ChangeValue);
RorBar.setOnSeekBarChangeListener(ChangeValue);
```

Simularly we'll make a button onclick listener with
```java
Button.OnClickListener ClickList = new Button.OnClickListener(){

      @Override
      public void onClick(View v) {

      }
  };
  ```
And subscrube our button to it with  `ConBtn.setOnClickListener(ClickList);` in onCreate, we'll also put the following code in to set our sliders and buttons to the default state.
```java
        RorBar.setProgress(50);
        ConBtn.setText("Connect to Arduino");
```

Just to make sure you are following this is our current code. (Your package name may very well be diffrent)

```java
package tofx.marius.arduinotcpclient;

import android.support.v7.app.ActionBar;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.SeekBar;

public class MainActivity extends AppCompatActivity {

    SeekBar MotBar;
    SeekBar RorBar;
    Button  ConBtn;
    Boolean Connected = false;
    Socket socket;
    OutputStream socketstream;
    PrintWriter output;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ActionBar actionBar = getSupportActionBar();
        actionBar.hide();
        getWindow().getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                |View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                | View.SYSTEM_UI_FLAG_FULLSCREEN
                | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);

        MotBar = (SeekBar)findViewById(R.id.Mot);
        RorBar = (SeekBar)findViewById(R.id.Ror);
        ConBtn = (Button)findViewById(R.id.Con);
        MotBar.setOnSeekBarChangeListener(ChangeValue);
        RorBar.setOnSeekBarChangeListener(ChangeValue);
        ConBtn.setOnClickListener(ClickList);
        RorBar.setProgress(50);
        ConBtn.setText("Connect to Arduino");        
    }

    SeekBar.OnSeekBarChangeListener ChangeValue = new SeekBar.OnSeekBarChangeListener() {
           @Override
           public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {

           }

           @Override
           public void onStartTrackingTouch(SeekBar seekBar) {

           }

           @Override
           public void onStopTrackingTouch(SeekBar seekBar) {

           }
       };

       Button.OnClickListener ClickList = new Button.OnClickListener(){

             @Override
             public void onClick(View v) {

             }
         };

}
```
Now would be a good time to check that everything works, we load the app onto our phone and drag the sliders and click the button and see that nothing changes but nothing crashes either so everything works fine. However we do notice that our app rotates and due to the nature of our app we really dont want it to do that so we'll disable that now by adding `setRequestedOrientation (ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);` to our onCreate(). Now when we debug the app agian we can see no rotation.

### Implimenting the TCP client finally
First our app will need the INTERNET permission, open `anifests/AndroidManifest.xml` and add `<uses-permission android:name="android.permission.INTERNET" />` inside `<application` so it looks like this

```xml
    ...<application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"

        android:theme="@style/AppTheme">
        <uses-permission android:name="android.permission.INTERNET" />
            <activity android:name=".MainActivity">...
```



In our button lisener we'll change our`onClick(View v)` to look like the following code
```java
@Override
      public void onClick(View v) {
          Thread connectthread = new Thread(){
              @Override
              public void run() {
                  socket = new Socket();
                  try {
                      Connected = true;

                      socket.connect(new InetSocketAddress("192.168.4.1", 1336), 1000);
                  }
                  catch (IOException e) {
                      Connected = false;
                  }
              }
          };
          connectthread.start();
          while (connectthread.isAlive()){}
            if (Connected){
                        ConBtn.setText("Connected!");
                        try {
                            socketstream = socket.getOutputStream();
                        } catch (IOException e) {
                            Connected = false;
                            ConBtn.setText("Failed to connect, are you on the WIFI?");
                        }
                    } else {
                        ConBtn.setText("Failed to connect, are you on the WIFI?");
                    }
                  }
      }
```
I'll now walk trough this code.
As a procaution in case we are not connected to the WIFI we will try the connect the socket inside, as if we ware to try this outside our intire app would crash due to the thread hanging up. We change our state to connected by changing the COnnected boolean to true, in our IOEXepction catch (which will be fired if connection failed) we make the boolean false. we then start the thread and use the while function to wait for the thread to finnish working, then we'll update the Button's text with the approrpiate text, we also initisate socketstream with the stream from our socket.

We are now connected! If we open up the Arduino Serial monitor we'll see a connection alert.

### Finnishing up the android app
Now we'll edit our Seekbar liseneer to look like the following:

```java
@Override
        public void onStartTrackingTouch(SeekBar seekBar) {
            if (output == null && Connected){
                output = new PrintWriter(socketstream, true);
            }
        }

        @Override
        public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
            if (output != null && Connected){
                new Thread(){
                    @Override
                    public void run() {
                        String payload = "{\"Ror\":"+RorBar.getProgress()+",\"Motor\":"+MotBar.getProgress()+"}";
                        output.println(payload);
                    }
                }.start();
            }

        }

        @Override
        public void onStopTrackingTouch(SeekBar seekBar) {

        }
```

When we first move our sliders we create a printwriter  with
```java
      if (output == null && Connected){
          output = new PrintWriter(socketstream, true);
      }
```

Now for the crown jewl, the icing on the cake, the code that sends data when we change the sliders value:
```java
@Override
        public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
            if (output != null && Connected){
                new Thread(){
                    @Override
                    public void run() {
                        String payload = "{\"Ror\":"+RorBar.getProgress()+",\"Motor\":"+MotBar.getProgress()+"}";
                        output.println(payload);
                    }
                }.start();
            }
        }
```
Again we need to put send data from a thread to prevent crashing, we first make a string and then send it with output.println(payload); if you check your console you'll see the values showup, dont worry about the slight corruption in the text this is caused by the diffrent band rates.

### Final android

MainActivity.java
```java
package tofx.marius.arduinotcpclient;

import android.content.pm.ActivityInfo;
import android.support.annotation.BoolRes;
import android.support.v7.app.ActionBar;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.SeekBar;

import java.io.BufferedWriter;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.ConnectException;
import java.net.Inet4Address;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.net.Socket;

public class MainActivity extends AppCompatActivity {

    SeekBar MotBar;
    SeekBar RorBar;
    Button  ConBtn;
    Boolean Connected = false;
    Socket socket;
    OutputStream socketstream;
    PrintWriter output;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ActionBar actionBar = getSupportActionBar();
        actionBar.hide();
        getWindow().getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                        | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                        | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                        | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                        | View.SYSTEM_UI_FLAG_FULLSCREEN
                        | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);

        MotBar = (SeekBar)findViewById(R.id.Mot);
        RorBar = (SeekBar)findViewById(R.id.Ror);
        ConBtn = (Button)findViewById(R.id.Con);
        MotBar.setOnSeekBarChangeListener(ChangeValue);
        RorBar.setOnSeekBarChangeListener(ChangeValue);
        ConBtn.setOnClickListener(ClickList);
        RorBar.setProgress(50);
        ConBtn.setText("Connect to Arduino");
        setRequestedOrientation (ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
    }

    SeekBar.OnSeekBarChangeListener ChangeValue = new SeekBar.OnSeekBarChangeListener() {


        @Override
        public void onStartTrackingTouch(SeekBar seekBar) {
            if (output == null && Connected){
                output = new PrintWriter(socketstream, true);
            }
        }

        @Override
        public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
            if (output != null && Connected){
                new Thread(){
                    @Override
                    public void run() {
                        String payload = "{\"Ror\":"+RorBar.getProgress()+",\"Motor\":"+MotBar.getProgress()+"}";
                        output.println(payload);
                    }
                }.start();
            }
        }

        @Override
        public void onStopTrackingTouch(SeekBar seekBar) {

        }

    };


    Button.OnClickListener ClickList = new Button.OnClickListener(){
        @Override
        public void onClick(View v) {
            Thread connectthread = new Thread(){
                @Override
                public void run() {
                    socket = new Socket();
                    try {
                        Connected = true;
                        socket.connect(new InetSocketAddress("192.168.4.1", 1336), 1000);
                    }
                    catch (IOException e) {
                        Connected = false;
                    }
                }
            };
            connectthread.start();
            while (connectthread.isAlive()){}
            if (Connected){
                ConBtn.setText("Connected!");
                try {
                    socketstream = socket.getOutputStream();
                } catch (IOException e) {
                    Connected = false;
                    ConBtn.setText("Failed to connect, are you on the WIFI?");
                }
            } else {
                ConBtn.setText("Failed to connect, are you on the WIFI?");
            }
        }

    };


}
```

Androidmanifest.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="tofx.marius.arduinotcpclient">

    <uses-permission android:name="android.permission.INTERNET" />
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

        </activity>

    </application>

</manifest>
```


You can also [download the android code I wrote](https://www.mediafire.com/?vkqh6bdgegd4kgo) if the android part of this handin was hard to follow.
---
