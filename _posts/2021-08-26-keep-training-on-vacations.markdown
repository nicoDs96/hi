---
title: "Keep training on vacations"
layout: post
date: 2021-08-26 10:23
tag: training
image: /assets/images/keep-training/
headerImage: true
projects: true
hidden: false # don't count this post in blog pagination
description: "I didn't want to buy a gym timer and I wasn't satisfied with digital ones. I ended up building one myself and I did it trying to increase my software engineering skills."
category: project
author: nicods
externalLink: false
---

<img class="image" src="{{ site.url }}/assets/images/keep-training/cover.png" alt="Cover Image"/>
In my free time, one of the activities I enjoy the most is (kick)boxing. I found an amazing gym in my hometown (thanks to Covid I left Rome, keep working remotely, and it was great!) that makes me more passionate than ever about fighting sports.

Very often a constant in fighting sports gyms is a semi-programmable timer that marks the seconds between training and rest. A common use for this timer is to emulate fight timing and repeat series of 3 min workouts and 1 min rest for 12 times more or less. In August I was on vacation and I wanted to keep training doing circuits based on time but how? I didn't want to buy a gym timer and I wasn't satisfied with digital ones. I ended up building one myself and I did it trying to increase my software engineering skills.</p>

<img class="image" src="{{ site.url }}/assets/images/keep-training/timer.jpg" alt="Gym Timer Picture"/>


## Where To Find It
You can fine the project on GitHub: [Project Repo](https://github.com/nicoDs96/BoxeTimer).

You can use it at: [The Timer Web App](https://nicods96.github.io/BoxeTimer/).

Feel free to open an issue or a pull request if you want to contribute. 

## Requirements
 I want to build something that:

* was able to alternate workout time and rest time like at the gym;
* was simple and small in size;
* can run on most mobile devices;
* write less code possible since I also want to enjoy my holiday;
* experiment things that I learned in theory but never practiced with;

## Design
 To satisfy those requirements I choose to build a web app hosted on Github since no backend logic was required and there is no effort nor cost to host a static page on [GitHub Pages](https://pages.github.com/).

I decided to use plain HTML + Javascript to keep it small and fast to load. The only library I used is [Bootstrap](https://getbootstrap.com/) to display the app properly on any device (both pc or mobiles).

The main components of the app are a Timer, that works as a clock and keeps track of time, and a Session, that defines the training and rest times and the number of repetition of the train-rest loop. 

### Tools and Project Setup

Since the main goal was to keep evrything simple I used [VsCode](https://code.visualstudio.com/) as editor since I had it on my pc but I never really used it without any specific estention. The project was organized in directories as follows:
```
|-- README.m
|-- ToDO.md
|-- index.html
|-- css
|   |-- FontImport.css
|   |-- bootstrap
|   |-- font
|   |   |-- DSEG7Modern-Regular.ttf
|   |   |-- DSEG7Modern-Regular.woff
|   |   `-- DSEG7Modern-Regular.woff2
|   `-- main.css
|-- js
|   |-- bootstrap
|   `-- src
|       |-- Controller
|       |   `-- TimerController.js
|       |-- Model
|       |   |-- Session.js
|       |   `-- Timer.js
|       |-- View
|       |   `-- ViewUtils.js
|       `-- utils
|           |-- WakeLocker.js
|           `-- zip.js
`-- res
    `-- buzz.mp3d
```
We have an index.html file where the UI is defined and where all the js scripts are referenced. There is the css folder where it is included the css boundles of the bootstrap library, a custom font and the main page colors (main.css). Then the js folder that contains all the application logic. Ther folder is divided into Model, View, Controller and utils folders to reflect what each script inside a folder is responsible for.

### Version Control Style
Of course, since the project is hosted on GitHub Pages, it must use some sort of version control. There are a myriad of git branching models ([GitFlow](https://nvie.com/posts/a-successful-git-branching-model/), [GitHubFlow](https://guides.github.com/introduction/flow/), etc ...) but I was curious to experiment with [Trunk Based Development](https://trunkbaseddevelopment.com/) that looks like the leading industry standard even if I was alone and I cannot really enjoy the benefits of the method.

<img class="image" src="{{ site.url }}/assets/images/keep-training/tbd.png" alt="Trunk Based VC"/>

The main page describes the method as:
> A source-control branching model, where developers collaborate on code in a single branch called ‘trunk’ *, resist any pressure to create other long-lived development branches by employing documented techniques. They therefore avoid merge hell, do not break the build, and live happily ever after.

In practice, all the developers work to a release on a release branch and when it is ready to go in production it is merged into master. In this specific case, a merge into master is equal to publishing on the webserver a new version of the web app.

In conjunction, each release is tagged with a release version number following the [semantic versioning standard](https://semver.org/), where the version number is given from a dot-separated triple MAJOR.MINOR.PATCH. Since we are working on a preview version the current releases are only MAJOR.MINOR because no patch is needed. 

### Observer Pattern

One of the major design focuses was trying to make everything asynchronous and event-based. It can be done easily with [DOM events](https://developer.mozilla.org/en-US/docs/Web/Events/Event_handlers) because it is what javascript is born for but it became more difficult when must be applied to custom objects and their state.

<img class="image" src="{{ site.url }}/assets/images/keep-training/obs.png" alt="Observer Pattern"/>

A solution to make the code asynchronous and to implement components that react upon state changes is the [Observer Pattern](https://en.wikipedia.org/wiki/Observer_pattern). It is similar to publish-subscribe communication models but implemented for objects: a component interested in an object state adds itself to the object's observers. The object, whenever its state changes, notifies all its observers of its new inner state. The components interested in the state change will react accordingly.

### Model View Controller

<img class="image" src="{{ site.url }}/assets/images/keep-training/mvc.png" alt="Observer Pattern"/>

 Quoting [Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) 
 > "Model–view–controller (usually known as MVC) is a [software design pattern](https://en.wikipedia.org/wiki/Software_design_pattern) commonly used for developing [user interfaces](https://en.wikipedia.org/wiki/User_interface) that divide the related program logic into three interconnected elements. This is done to separate internal representations of information from the ways information is presented to and accepted from the user. [...] Popular programming languages have MVC frameworks that facilitate implementation of the pattern."

Since almost all the frameworks to build something with a GUI implements MVC under the hood, I wanted to experiment with applying MVC by myself without any framework or tool to facilitate the work. 

## Implementation
Before deep-diving into code, let's talk about how the problem is modeled. We have two main classes Timer.js and Session.js. They both implement the observer pattern: Session objects observe Timers and GUI controller observes Sessions.  

A Timer object has the only ability to count seconds (or fractions of seconds) depending on the input. Each time its inner state is updated (the number of seconds elapsed from the last time is increased) all the observers are notified.  

{% highlight js %}
class Timer{  


    constructor(updateInterval = 500) {
        this._seconds = 0.0 
        this._observers = []
        this._updateInterval = updateInterval
        this._updater = null
    }
    update() {
        this._seconds += this._updateInterval / 1000 
        
        this._observers.forEach(observer => {
            observer.notify( this.getSeconds() )
        });
    }
    start() {
        this._updater = setInterval( 
            (timer = this) => {timer.update()}
            , this._updateInterval
        )
    }
    stop() {
        clearInterval( this._updater )
    }
    getSeconds() {
        return this._seconds
    }
    setObservers(observersArrays) {
        this._observers = observersArrays
    }
    addObserver(observer) {
        this._observers.push(observer)
    }
    addObserverArray(observers) {
        observers.forEach( observer => {
            this._observers.push(observer)
        })
    }
}
{% endhighlight %}

The class has main control methods like start(), stop(), update(), and utility methods to add observers or get inner state. In detail a Timer object is built with an "update interval", telling the class to update the time and notify other objects each X milliseconds. The timer exploits the javascript utility [setInterval](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setInterval) and each time the setIntervall callback is invoked we update the timer seconds count and notify the observers.

### Observable Session
Session is initialized with an array of seconds and a number of repetitions. Our goal is to count from 0 to the upper bound in the array for each entry (upper bound) in the array and repeat for the specified number of repetitions. Each time we reach an upper bound a buzzer makes a sound to notify the people using the app.  

Session objects observe a Timer object and thus implement the notify method. The first time notify is invoked the session object memorizes the timerSeconds. Then on the next calls it is able to tell how many seconds are elapsed and react accordingly. A session is also observable since the GUI controller will observe it to update the graphical components.  

{% highlight js %}
Session{

constructor(timesArray, repetition, buzzerPath = 'res/buzz.mp3') {
        this._timesArray = timesArray
        this._repetition = repetition
        this._buzzer = new Audio(buzzerPath)
           
        this._currentElapsedTime = 0.0
        this._prevTime = 0.0
        this._currentRepetition = 0
        this._currentTimeIndex = 0 
        this._isStopped = false


        this._observers = []
    }


... 

notify(timerSeconds){ 


        if(!this._isStopped){
            if(this._prevTime == 0) {this._prevTime = timerSeconds} //only for the firs update


            this._currentElapsedTime += (timerSeconds - this._prevTime)
            this._prevTime = timerSeconds
            
            if(this._currentElapsedTime >= this._timesArray[this._currentTimeIndex]){
                this.playSound() //buzz
                // switch to next time
                this._currentTimeIndex = (this._currentTimeIndex + 1) % this._timesArray.length 
                
                // when all the times are elapsed
            
                if(this._currentTimeIndex == 0){  
                    this._currentRepetition += 1 //update repetition
                }
                
                //reset current time when switching to a new time limit
                this._currentElapsedTime = 0.0           
            }
    
            if(this._currentRepetition == this._repetition){
                this.stop()
            }
                    
            this._observers.forEach(observer => {
                observer.notify( this.getState() )
            });
        }
 
...
{% endhighlight %}


### Prevent Screen Lock 

 Since the training sessions should be longer than the time our screen stays awake and since a web app requires the page to be always on focus to work properly, we should find a way to prevent device screen lock to bother our training sessions. After some intense googling I came up with [Screen Wake Lock API](https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API). It allows us to prevent device screen lock. Exactly what I was looking for. Unfortunately, it is not supported by all mobile browsers for now so it reduces the support of our web app. However devices with Google Chrome 84 or greater are ok, as well as some others listed in the [MDN API](https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API) doc page, which is better than nothing.

The procedure is only copy-paste code from MDN documentation and simply it locks the screen when we load the page, we re-lock the screen whenever the page loses and then acquire focus again. 

{% highlight js %}
if ('wakeLock' in navigator) {
    isWakeLockSupported = true;
    console.info('Screen Wake Lock API supported!')
} else {
    isWakeLockSupported = false;
    console.error('Wake lock is not supported by this browser.')
}


let wakeLock = null;


let lock = async () =>{
  try {
    wakeLock = await navigator.wakeLock.request('screen');
    console.info('Wake Lock is active!')
  } catch (err) {
    console.error(`The Wake Lock request has failed - usually system related, such as battery:\n${err.name}, ${err.message}`)
  }
}


document.addEventListener('visibilitychange', async () => {
  if (wakeLock !== null && document.visibilityState === 'visible') {
    wakeLock = await navigator.wakeLock.request('screen');
    console.info('Wake Lock is active again.')
  }
});


lock()
{% endhighlight %}


### Timer GUI
__View__: The View by definition is what the user can see. In our case, it is the simulation of a 7 segment display and some input and buttons to control it. To make everything simple all the GUI elements are in index.html and are plain HTML tags. All the control, i.e. behavior of the components is defined inside the controller.

All the buttons are not represented with icons or button elemetes. Instead they are simple Unicode strings styled using colors and font size in css. This allows to maintain the page lightweight but yet looking nice.

__Controller__: the controller (TimerController.js) is responsible for: 
*  dynamic creation of elements in javascript, like when we click on + or - buttons;
* react to Session object state changes, like highlight the current time or the current repetition number;
* Update GUI elements like displayed time;
* Add logic to clickable element;
* Initialize the page before the training session is set up and started.

## Conclusions 
This project helped me to stay trained on vacation and to improve my software engineering skills by practicing with things at low level (no framework, implement almost evrything from scratch). I have learned:
* How to implement MVC
* How to implement Observer Pattern
* To design a web app
* That exist a Screen Lock API specification that later or sooner will be implemented in all the modern browsers. This might mean nothing but I think it is a step forward to get rid of app stores, that i hate by the way (but that's another story ...)
* New way of using version control
* How to corerctly use semantic versioning (YES, there is a specification!)

 and above all, if you do not like what you find around do not be satisfied, get your hands dirty and improve it! At worst you will have broadened your knowledge.

Disclaimer note: the Session class is not complete on its functionalities and in the future I would like to add Session that can be paused. This might result into wider applicaiton for the timer, like in a basketball game for example.

If you want to contribute, the repo contains a toDO.md file, keep a task, open a issue to discuss the feature and maybe pull the feature. 

## References


* [The Timer Web App](https://nicods96.github.io/BoxeTimer/)
* [Project Repo](https://nicods96.github.io/BoxeTimer/)
* [GitHub Pages](https://pages.github.com/)
* [Bootstrap](https://getbootstrap.com/)
* [Visual Studio Code](https://code.visualstudio.com/)
* [GitFlow](https://nvie.com/posts/a-successful-git-branching-model/)
* [GitHubFlow](https://guides.github.com/introduction/flow/)
* [Trunk Based Development](https://trunkbaseddevelopment.com/)
* [Semantic Versioning](https://semver.org/)
* [Model View Controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)
* [Screen Wake Lock API](https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API)

