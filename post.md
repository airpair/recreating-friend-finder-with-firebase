## Firebase Friends

Do you remember Boost Mobile? Like a lot of the early 2000s, it's rescinded into the shrouds of the past. But I always thought its friend finder feature, which would tell you where your friends were, was very cool. 

![Boost Friend Finder](http://www.blogcdn.com/www.engadget.com/media/2006/09/boost-loopt-screencap.jpg)

Lets recreate it!

What do we need to build?
- A way to supply the location of the user.
- A backend service to register all of our users and keep their current location. (Authentication and Authorization for our users)
- A front end visualization that will display the location.

In this tutorial we'll be using a couple of different services to accomplish this. Since we're going to want to use this from a mobile platform (since that's when our friends will be moving) we can use <a href="http://phonegap.com/">Phonegap </a>and the <a href="http://docs.phonegap.com/en/edge/cordova_geolocation_geolocation.md.html">Geolocation API</a> to provide updates on the current location. To store the location, <a href="http://www.firebase.com">Firebase</a>, an awesome Backend as a Service, will let us easily synchronize data. And Google Maps will visualize the locations of our friends on these maps.

## Geolocation
For our service we need to have our friends download a client program that they can use to update the GPS coordinates. To start let's build a very simple one page app, with a button to activate the call to the Geolocation API.

```javascript,linenums=true
//Geolocation call

function onSuccess(position) {
    var lat = position.coords.latitude;
    var longitude = position.coords.longitude; 
   
    //Stuff to do with latitude and longitude
}

function onError(e){
    alert('Error: ' + e.message + '\n' + 'Not able to establish gps');
}

var geoPosition = navigator.geolocation.watchPosition(onSuccess, onError, { timeout: 60000 });

```

This code is doing a couple of things. Reading from bottom to top, we're calling navigator.geolocation, the <a href="http://docs.phonegap.com/en/edge/cordova_geolocation_geolocation.md.html">API</a> to read the current location from the mobile phone. Now we've got a couple of methods we could call on this. We could use the 'getCurrentPosition' to return a specific location, one time. But since we're expecting this to serve as a self reporting 'tracker', we don't want to have to force our friends to return to the app each time to press the button. So let's use the watchPosition() to return the latitude and longitude of the mobile phone every 60 seconds. If there's an error, it'll be passed into the onError(e) function, which will alert with a message. If the phone successfully gets the coordinates, onSuccess() will take the position object, and save lat and long.

```
<!DOCTYPE html>
<html>
  <head>
    <title>Geolocation</title>
    <meta name="viewport" content="initial-scale=1.0, user-scalable=no">
    <meta charset="utf-8">
<script type="text/javascript">
</script>
function shareLocation(){

      function onSuccess(position) {
        var lat = position.coords.latitude;
        var longitude = position.coords.longitude; 
   
        console.log("lat: " + lat + ", long: " + longitude );
      }

      function onError(e){
        alert('Error: ' + e.message + '\n' + 'Not able to establish gps');
      }

      var geoPosition = navigator.geolocation.watchPosition(onSuccess, onError, { timeout: 60000 });
    }

</head>
<body>
      <button onclick="shareLocation()">Share Location</button>
</body>
</htm>
```

##Firebase
Now we need to do something with those locations. Firebase is a Backend as a Service, an extremely easy way to store and retrieve data. The <a href="https://www.firebase.com/docs/">Firebase Docs</a> are great, read through it, but tl;dr you can push data with Firebase.Push as an array, and Firebase.Set to save data at a particular location. We could structure our app in a couple of different ways. We could have an array, where everytime the callback is called it will push the latitude and longitude to an array.

It would look a little something like this.

```javascript,linenums=true

[
    {lat: 000, long: 000},
    {lat: 001, long: 000},
    {lat: 002, long: 000},
    ...
]
```

We could use that to track locations over time, determine popular locations, and visualize paths. But lets keep it simple. One location at any one time for any one user, using Firebase.set.

Whenever the value changes, the callback will trigger the update and we can set the new value. 

```javascript,linenums=true
var ref = new Firebase('YOUR FIREBASE');

function onSuccess(position) {
    var lat = position.coords.latitude;
    var longitude = position.coords.longitude; 
   
    ref.set({mylat: lat, mylong: longitude});
}
```



Lets check out the firebase schema:
```javascript,linenums=true
{
    mylat: 1,
    mylong: 1
}
```

We're setting the values at the root node. If multiple people were doing this, they would overwrite one another. We need to give each user their own place to set values.

##Authentication

Firebase has a built in Authentication method, that will let us give users a way to authenticate. We can use the <a href="https://www.firebase.com/docs/web/guide/user-auth.html">Simple Login</a> method.
First go to your particular firebase and, in the authentication section, turn on SimpleLogin.
Now lets modify our HTML page, and add in two input boxes for email and password.

```
      <input type = "email" id="emailLogin" placeholder="Email" />
     <input type = "password" id="password" placeholder="Password"/>
```

Make sure you have jQuery added to the page, it will come in handy to pull values from those boxes.


```javascript,linenums=true

  function loginUser(emailLogin, password){ 
      var emailLogin = $("#emailLogin").val();
      var password = $("#password").val();
        ref.authWithPassword({
            email: emailLogin,
            password: password
          }, function(error, authData){
            if (error) {
                console.log("Login Failed!", error);
             } else {
              console.log('yes');
               userRef = ref.child("users").child(authData.uid);
              }
                    
          });
    }

  function createUser(emailLogin, passwordLogin){
    var emailLogin = $("#emailLogin").val();
    var password = $("#password").val();
    ref.createUser({
      email: emailLogin,
      password: password
    }, function(error, user) {
     if (error === null) {
        console.log("User created successfully");
          var userNew = {email: emailLogin, mylat: 0, mylong: 0};
          userRef = ref.child("users").child(user.uid);
            userRef.set(userNew);
          } else {
            console.log("Error creating user:", error);
          }
        });
      }
```
There are two separate functions, loginUser and createUser. The createUser function is reading in the email and password values, and calling the createUser
method on the firebase reference. This looks for an email and a password, and on success we create a new object, userNew, and set that at ref.child('users').child(user.uid), which will be a unique node for each of our friends. 


```javascript,linenums=true
{
  "users" : {
    "simplelogin:1" : {
      "email": "testing@email.com",
      "mylat" : 1,
      "mylong" : 1
    },
    "simplelogin:2" : {
      "email": "testing2@email.com",
      "mylat" : 5,
      "mylong" : 1
    }
  }
}

```
And as you can see we're now setting the values on the child node. We've got a user, and under it we're pushing the latitude and longitude data.

The login user functionality is very similar, but instead of creating the refence point if uses the UID to find each user's unique node, and returns a reference to that location,
so that updates to the location can be set for the correct user.

Here is our code so far:

```javascript,linenums=true
//Initial firebase
var ref  = new Firebase("https://boostfriend.firebaseio.com/");

//Reference to the userData location
var userRef;

function shareLocation(){
function onSuccess(position) {
        var lat = position.coords.latitude;
        var longitude = position.coords.longitude; 
   
        userRef.set({mylat: lat, mylong: longitude});
      }

      function onError(e){
        alert('Error: ' + e.message + '\n' + 'Not able to establish gps');
      }

      var geoPosition = navigator.geolocation.watchPosition(onSuccess, onError, { timeout: 60000 });
    }

  function loginUser(emailLogin, password){ 
      var emailLogin = $("#emailLogin").val();
      var password = $("#password").val();
        ref.authWithPassword({
            email: emailLogin,
            password: password
          }, function(error, authData){
            if (error) {
                console.log("Login Failed!", error);
             } else {
              console.log('yes');
               userRef = ref.child("users").child(authData.uid);
              }
                    
          });
    }

  function createUser(emailLogin, passwordLogin){
    var emailLogin = $("#emailLogin").val();
    var password = $("#password").val();
    ref.createUser({
      email: emailLogin,
      password: password
    }, function(error, user) {
     if (error === null) {
        console.log("User created successfully");
          var userNew = {email: emailLogin, mylat: 0, mylong: 0};
          userRef = ref.child("users").child(user.uid);
            userRef.set(userNew);
          } else {
            console.log("Error creating user:", error);
          }
        });
      }
  }
```
*we've got some stuff to improve, like the repeated reference to the input boxes in both the login and create user function, but lets save it for our refactor after we ship and make millions.

##Authorization

We need to make sure that our friends can't impersonate one another, and that they can only set data to their unique ID. Firebase provides a set of <a href="https://www.firebase.com/docs/security/guide/">security rules</a> you can define per firebase to regulate what can and cannot be modified by certain users (as well as many other things ex. validation of data).

There are a lot of different ways to define access and authorization rules for Firebase, but luckily we've got a pretty simple situation. We want our friends to be able to read everyone's location that is provided, and we want them to only be able to write location to their specific node.

```javascript,linenums=true
{
  "rules": {
    ".read": true,
    "users": {
      "$user_id": {
        ".write": "$user_id === auth.uid"
      }
    }
  }
}
```
Because we're using the built in firebase login method, we'll have access to auth.uid. And the $user_id is a flexible expression that captures whatever is at that location in our Firebase schema. So after a user creates an account, it saves their UID to that location, and when any user tries to write data to that location/anything under that location, it will check to make sure the currently logged in user is the same one as the one who created the account.

##Dashboard: The Map

Now we have  a lot of data from different clients, and our friends can log in and share their location. But now we want to see where everyone is. We can take the different points and visualize it using the Google Maps API. This code from the Google Maps team

https://developers.google.com/maps/documentation/javascript/examples/map-geolocation 

is what we will be duplicating. We want each of our friends locations to be created as a marker on a map, and when their location is updated the marker should similarly
update.

We need to start by adding a place for the map to be bound to on the page

```
<div id="map-canvas"></div>
```

and the default code to initialize it:
```javascript,linenums=true
function initialize() {
  var mapOptions = {
    zoom: 1
  };
  map = new google.maps.Map(document.getElementById('map-canvas'),
      mapOptions);

  // Try HTML5 geolocation


      var pos = new google.maps.LatLng(0,
                                       0);
      map.setCenter(pos);
}
```

Now that we have bunch of data in firebase, we should read it out and create indicators for each of our friends.
```javascript,linenums=true
function getEveryonesData(ref){
  ref.on('value', function(data){
        data.forEach(function(el){
          createFriendMarker(el.val());
        }
      )
  });
}
```

This takes a reference to firebase, and on a value change it will iterate through each of the users in our user JSON and call a function, createFriendMarker, on those user objects.

```javascript,linenums=true
function createFriendMarker(user){
  var friendPos = new google.maps.LatLng(user.mylat, user.mylong);
  var friendMarker = new google.maps.Marker({
    position: friendPos,
    map: map,
    title: user.email
  });
  markers.push(friendMarker);
}
```

This initializes a new <a href="https://developers.google.com/maps/documentation/javascript/reference#LatLng">LatLng object</a>, and uses the info from the user object as the latitude and longitude. Then it creates a new <a href="https://developers.google.com/maps/documentation/javascript/reference#Marker">Marker</a>, with the position
as that LatLng, and the map as the current map.  

We also push it to an array that will hold all of our markers, to make it easy to reference and change them. If we were to run this now, every time a value updates
wed get a new marker, without removing the older one. If we add a function to clear the markers in the array we can remove all of them on every value update.

```javascript,linenums=true
function clearMarkers(){
  for (var i = 0; i < markers.length; i++) {
    markers[i].setMap(null);
  }
}
```

Here is our code for updating the Google Map.

```javascript,linenums=true
var map;
var markers = [];

function clearMarkers(){
  for (var i = 0; i < markers.length; i++) {
    markers[i].setMap(null);
  }
}


function createFriendMarker(user){
  var friendPos = new google.maps.LatLng(user.mylat, user.mylong);
  var friendMarker = new google.maps.Marker({
    position: friendPos,
    map: map,
    title: user.email
  });
  markers.push(friendMarker);

}

function getEveryonesData(ref){
  ref.on('value', function(data){
      clearMarkers();
        data.forEach(function(el){
          createFriendMarker(el.val());
        }
      )
  });
}

function initialize() {
  var mapOptions = {
    zoom: 1
  };
  map = new google.maps.Map(document.getElementById('map-canvas'),
      mapOptions);

  // Try HTML5 geolocation


      var pos = new google.maps.LatLng(0,
                                       0);

      map.setCenter(pos);

      
}

google.maps.event.addDomListener(window, 'load', initialize);
```

##Making it Mobile
Of course we want to be able to bundle this for our phone, so that when we're on the go it can update accordingly. 

I like the <a href="https://build.phonegap.com">Phonegap Build </a> service - it's a great way to quickly and rapidly prototype a mobile app. Put in an existing open source github repo, and it will pull the website and convert it into a native apk you can download for your mobile phone.

Here is the <a href="https://github.com/ghabs/boostfriend/blob/master/index.html">complete code</a>. The additional modifications are adding a css file, ionic.css, to add mobile friendly styling, and adding an additional button that will stop sharing our location.

##Extensions

There is obviously a lot more we can do with this. For one it's very user unfriendly. Forcing our friends to type in their username and password every time is a sure fire way to lose friends. Firebase provides login methods more suitable to a mobile environment, like Google+ or Facebook connections, which would give us access to social features that could let us do some very cool stuff. Plus we've built all of this with standard HTML and JS - we could extend this with AngularJS or React and get a lot of additional functionality. But for the sake of relieving the golden days of Boost Mobile commercials I'm satisfied. 








