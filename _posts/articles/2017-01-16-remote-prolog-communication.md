---
layout: article
title: Remote Prolog communication
meta: In this article you will learn how to use speech recognition and communicate with Prolog by voice
category: articles
author: Tadeusz Lewandowski
author_page: https://github.com/tadeusz-lewandowski
---

Recently I got a very interesting task at the university - use your phone to communicate with Prolog. Sounds like a simple AI programming so let's code that!

## Prolog

In Prolog we write **knowledge base** (facts and relations) - and **queries** (questions to this knowledge base).
Let's begin with writing some facts about the weather.

* today will be sunny day
* tomorrow will be windy and rainy day
* if some day will be windy and rainy then this will be cloudy day

We can code this facts in Prolog like this

``` prolog
sunny_day(today).
windy_day(tomorrow).
rainy_day(tomorrow).

cloudy_day(X) :- windy_day(X), rainy_day(X).

```

Save this in the file *knowledge.pl*
Ok, now Prolog knows some facts about the weather! The last line may be a bit confusing. You must read this like: the day will be cloudy **if** the day will be windy **and** rainy. The uppercased X means a day - it will be explained in the next step.

Now we can ask prolog some questions. If you have installed swi-prolog, open terminal (in Linux) and go to the place where you created your knowledge base. Run Prolog program with one argument - name of your knowledge base.

``` bash
prolog knowledge.pl
```

This command runs Prolog and loads knowledge base automatically. Now you can give Prolog some questions! Check if today we'll be sunny day.

```
?- sunny_day(today).
```
Prolog says **true** because we wrote this fact in the knowledge base.
```
?- sunny_day(tomorrow).
```
Prolog says **false** because we didn't write this fact in the knowledge base. Prolog only knows that today will be sunny day.
We wrote universal prolog fact - cloudy_day - that is true only if the day will be windy and rainy. We don't have to hardcode facts about all days in calendar. The X will be day that we ask for. If we write query like this:
```
?- cloudy_day(tomorrow).
```
then X is tommorow. Prolog will check if this is true:
```
cloudy_day(tomorrow) :- windy_day(tomorrow), rainy_day(tomorrow).
```
See, Prolog set all Xs as tomorrow.
Exactly what we want! You can ask about any day, for example today, Friday and any day you want. This query will return true if you write proper facts about days.

## Mobile app
Our app must be able to:

* recognize voice
* send sentence to the remote server(server must understand this and give Prolog proper queries
)

I decided to write this project in JavaScript.

## Node.js
In short Node.js is JavaScript that runs on your local machine. It means that you can manage your file system (yeach, you can write viruses using Node ;) ) and even write http server in JS. We are going to use Node.js as our server that receive sentences, parse it and give as query to Prolog.

Let's code server!
Make sure you have installed Node and npm. Create directory for your project and open terminal in that place. Run
``` bash
npm init
```
This will initialize Node project and create package.json (package.json contains information about the libraries, project version, authors etc).

Create *index.js* file (*touch index.js* in Linux terminal) and open it in your favorite code editor (sublime in my case).

Npm it's a beautiful tool for many things like downloading necessary libraries. If you want to install something you don't have to manually download libraries. Npm does it for you. Here's how it looks like:
``` bash
npm install <something>
```
In our project we will write http server using **express** Node library. It's very popular http library in Node.js world. Install it via terminal:
```
npm install --save express
```
--save - saves express version information in package.json (very helpful for other developers)

Write basic server code in *index.js*. Commented lines explain what's going on.

``` javascript
// includes express library
var express = require('express')
// create express app
var app = express()

// send hello on / route
app.get("/", function (req, res) {
  res.send("Hello")
})

// listen on 3000 port
app.listen(3000, function () {
  console.log("'I'm listening on 3000!")
})

```
Run server using terminal:
``` bash
node .
```
App is running on 3000 port and serves "hello" message on / route. Open a browser and type in address bar *localhost:3000*. The browser do *GET request* to our server on / route. Server responds with the text "hello" which will appear in your browser.

Next step is to write express *get function* that will receive questions. We should pass *question* as a  GET parameter. So code it:

``` javascript
// includes express library
var express = require('express');
// create express app
var app = express();

// send hello on / route
app.get("/", function (req, res) {
  res.send("Hello")
});
// send user question on /api/anwser/
app.get("/api/anwser/:question", function(req, res){
    res.send(req.params.question);
});

// listen on 3000 port
app.listen(3000, function () {
  console.log("'I'm listening on 3000!");
});

```

When we type in browser *localhost:3000/api/anwser/mysentence* the server responds with "mysentence". It means that server has no problem receiving our question.

## Simply parser
In next step we write *question understanding* and mechanism of creating Prolog queries. Weather knowledge base is not complicated so I used only **regular expressions**. For more complex sentence parsing use Node.js libraries with language parsers.

Regular expressions will check whether our question match any **pattern** and if yes, create proper Prolog query. Lets code this:

*in question route*
``` javascript
app.get("/api/anwser/:question", function(req, res){
    var question = req.params.question;
    
    if(/^.* weather .*/.test(question)){
        var splitted = question.split(" ");
        var weather = splitted[0];
        var day = splitted[2];
        
        var prologQuery = "";
        if(weather == "cloudy"){
            prologQuery = "cloudy_day(" + day + ")"; 
        } else if(weather == "sunny"){
            prologQuery = "sunny_day(" + day + ")";
        } else if(weather == "windy"){
            prologQuery = "windy_day(" + day + ")";
        } else if(weather == "rainy"){
            prologQuery = "rainy_day(" + day + ")";
        }
        
        if(!prologQuery.length){
            res.send("I don't understand");
        } else{
            // here we need prolog
        }
        
    } else{
        res.send("I don't understand");
    }
});
```

The
``` javascript
/^.* weather .*/.test(question)
```
check if the question matches the pattern "**kind of weather** weather **day**". This regular expression will pass sentences like "sunny weather tomorrow" or "cloudy weather today".

If server recognized the weather then the Prolog query is created. In other case the server will respond "I don't understand" to user.

## Prolog implementation in Node.js
It's time to bring Prolog to the project. The easiest way is to use Prolog implementation in Node.js (it's awesome that someone did something like this).

The library is called *jsprolog*. Install it via terminal:
``` bash
npm install --save jsprolog
```
Next we need to require this library (exactly like express) on the top of script:
``` javascript
var Prolog = require('jsprolog');
```
Ok, we have Prolog object. If you remember we need knowledge base as well.
*jsprolog* can use common string variable that contains our base. 

See how to set knowledge base:

*first lines of script*
``` javascript
// includes express library
var express = require('express');
var Prolog = require('jsprolog');
// create express app
var app = express();

// set knowledge base text
var knowledgeString = "sunnyDay(today). windy_day(tomorrow). rainy_day(tomorrow). cloudy_day(X) :- windy_day(X), rainy_day(X).";

// set knowledge base (pass knowledgeString as an argument)
var db = Prolog.default.Parser.parse(knowledgeString);

// ...
// server requests, listen etc.
```
Come back to question parse. Now we can easly gives queries to Prolog object and show results:

*index.js - /api/anwser/:question method*
``` javascript
// ...
app.get("/api/anwser/:question", function(req, res){
    var question = req.params.question;
    if(/^.* weather .*/.test(question)){
        var splitted = question.split(" ");
        var weather = splitted[0];
        var day = splitted[2];
        
        var prologQuery = "";
        if(weather == "cloudy"){
            prologQuery = "cloudy_day(" + day + ")."; 
        } else if(weather == "sunny"){
            prologQuery = "sunny_day(" + day + ").";
        } else if(weather == "windy"){
            prologQuery = "windy_day(" + day + ").";
        } else if(weather == "rainy"){
            prologQuery = "rainy_day(" + day + ").";
        }
        
        if(!prologQuery.length){
            res.send("I don't understand");
        } else{
            // here we need prolog
            // this parse query string
            console.log(prologQuery)
            var query = Prolog.default.Parser.parseQuery(prologQuery);
            // this runs query
            var iter = Prolog.default.Solver.query(db, query);
            // return the next result of prolog query
            res.send(iter.next());
        }
        
    } else{
        res.send("I don't understand");
    }
});
// ...
```
You may be confused about "why iter.next() ?". You need that because Prolog can give you many results. For example try creating more sunny days and run query *sunny_day(X)*. The Prolog will output all days that will be sunny.

## Server - finished
We finished working on server. Try to send requests by browser or Postman (more comfortable approach). Full code of server:

*index.js*
``` javascript
// includes express library
var express = require('express');
// include jsprolog library
var Prolog = require('jsprolog');
// create express app
var app = express();

// set knowledge base text
var knowledgeString = "sunny_day(today). windy_day(tomorrow). rainy_day(tomorrow). cloudy_day(X) :- windy_day(X), rainy_day(X).";

// set knowledge base (pass knowledgeString as an argument)
var db = Prolog.default.Parser.parse(knowledgeString);

// send hello on / route
app.get("/", function (req, res) {
  res.send("Hello")
});
// send user question on /api/anwser/
app.get("/api/anwser/:question", function(req, res){
    var question = req.params.question;
    if(/^.* weather .*/.test(question)){
        var splitted = question.split(" ");
        var weather = splitted[0];
        var day = splitted[2];
        
        var prologQuery = "";
        if(weather == "cloudy"){
            prologQuery = "cloudy_day(" + day + ")."; 
        } else if(weather == "sunny"){
            prologQuery = "sunny_day(" + day + ").";
        } else if(weather == "windy"){
            prologQuery = "windy_day(" + day + ").";
        } else if(weather == "rainy"){
            prologQuery = "rainy_day(" + day + ").";
        }
        
        if(!prologQuery.length){
            res.send("I don't understand");
        } else{
            // here we need prolog
            // this parse query string
            console.log(prologQuery)
            var query = Prolog.default.Parser.parseQuery(prologQuery);
            // this runs query
            var iter = Prolog.default.Solver.query(db, query);
            // return the next result of prolog query
            res.send(iter.next());
        }
        
    } else{
        res.send("I don't understand");
    }
});

// listen on 3000 port
app.listen(3000, function () {
  console.log("'I'm listening on 3000!");
});
```

## Client app
The server gets requests via http and sends responses via http. It means that you can write client side app in any environment - Android studio, Xcode, Xamarin and many others because we don't send html templates. It's very flexible and you can write many clients for this one server (remember about cross origin requests, Node has library for that as well - *cors*).

I decided to use mobile browser as client side app. 

## Preparing server for sending html
We would like to use speech as communication tool. Web browsers offer us speech api. I tested web speech api in firefox and chrome but unfortunately this works only in chrome. Mozilla says that you must switch option in about:config but this is not working. Maybe this issue will be fixed soon.
So we use chrome. Another requirement is that html file must be sent from the server because web speech doesn't work when you just use it locally.
If you want to send static files in express you must define which directory will be storing public files. Create *public* directory (*mkdir public* in Linux terminal). Inside *public* create *index.html*.

Now tell express that we want *public* to be static directory (contains things like images, html templates and scripts):
*index.js - under line var app = express()*
``` javascript
// ...
app.use(express.static('public'));
// ...
```
Change old *hello* reply to
``` javascript
res.sendFile("index.html");
```
When you do get request on / the server respond with index.html file.

## Html, JS, Qwest

Oue goal is to send questions by speeking. First, lets create basic html template:
*index.html*
``` html
<!DOCTYPE html>
<html>
<head>
    <title>Speech Prolog</title>
    <meta charset="utf-8">
</head>
<body>
    <h1>Click on page to speek</h1>

    <script>
        
    </script>
</body>
</html>
```
We need library for sending questions to server.
**Qwest** is a great small library for http requests. Why not jQuery? Well, jQuery is huge and we are going to use only small part of it so better solution is to chose minimal library like Qwest.

Include qwest in *head* tag:
*index.html*
``` html
// ...
<script src="https://cdnjs.cloudflare.com/ajax/libs/qwest/4.4.5/qwest.min.js"></script>
// ...
```

Mechanism we need:

* when you click/tap on screen, the voice recognition starts
* when voice recognition ends, the sentence will be sent to the server
* server replies with message from Prolog and show in JS alert

Start with simply click event on body:
*index.html - inside script tag*
``` javascript
document.addEventListener("click", function(){
    
});
```
Chrome browser gives us **webkitSpeechRecognition** api for speech recognition. Speech recognition object has some interesting events - result and speech end. The first one is the most important because when we have results from recognition we want to send this to server. Then we use qwest with *get request*. Lets code that:
*index.html - inside script tag*
``` javascript
var recognition = new webkitSpeechRecognition();

recognition.addEventListener("result", function(event){
    var sentence = event.results[0][0].transcript;
    console.log(sentence)
    // qwest do get request for route /api/anwser/<something>
    qwest.get(`/api/anwser/${sentence.toString()}`, null, { cache:true })
    // do the function when server respond
    .then(function(xhr, response) {
        // display response
        alert(response);
    });
});

recognition.addEventListener("speechend", function(event){      
    recognition.stop();
});

document.addEventListener("click", function(){
    recognition.start();
});
```
Note that in Qwest request I used ES6 syntax for writing variables inside string. You can replace it by *"something/" + variable* if you want.

## Client app - finished
Now you can talk to the server! Just tap/click on screen and start talking. You must, of course, allow browser to use microphone before. Full code:
*index.html*
``` html
<!DOCTYPE html>
<html>
<head>
    <title>Speech Prolog</title>
    <meta charset="utf-8">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qwest/4.4.5/qwest.min.js"></script>
</head>
<body>
    <h1>Click on page to speek</h1>

    <script>
        var recognition = new webkitSpeechRecognition();

        recognition.addEventListener("result", function(event){
            var sentence = event.results[0][0].transcript;
            console.log(sentence)
            qwest.get(`/api/anwser/${sentence.toString()}`, null, { cache:true })
            .then(function(xhr, response) {
                alert(response);
            });
        });

        recognition.addEventListener("speechend", function(event){      
            recognition.stop();
        });

        document.addEventListener("click", function(){
            recognition.start();
        });
    </script>
</body>
</html>
```

## Conclusion
That was a very cool task to do. 
This is a very basic version of communication between mobile devices and Prolog. We created Node.js server that can respond to any devices that use http protocol and even send html file to browsers. Server has built-in Prolog implementation.  
Extend the code with more complex Prolog knowledge base, use better parser and have fun. 
If you have questions or you want to show your project to me, write at
tadeusz.m.lewandowski@gmail.com

Thanks for reading

Knowledge source:

[Prolog basics](http://lpn.swi-prolog.org/lpnpage.php?pagetype=html&pageid=lpn-htmlch1)
[Http requests](http://www.restapitutorial.com/lessons/httpmethods.html)
[Speech recognition](https://developer.mozilla.org/en-US/docs/Web/API/SpeechRecognition)
[Express](http://expressjs.com/)
[Qwest](https://github.com/pyrsmk/qwest)