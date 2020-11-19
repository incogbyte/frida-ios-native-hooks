# frida-ios-native-hooks
tracing with frida-trace some native apis or libs


1 - Tracing Encryption

```bash
$ frida-trace -i CCCrypt -U TargetApp
```

* edit scripts at __handlers__

```javascript
onEnter: function (log, args, state) {
    log('CCCrypt()');
    log('  ----- algo:' + args[1].toInt32());
    log('  ----- key:');
    log(hexdump(args[3], {length: args[4].toInt32()}));
    log('  ----- in iv:');
    log(hexdump(args[5], { length: 16 }));
    log('  ----- in data:');
    log(hexdump(args[6], {length: args[7].toInt32()}));
    this.args = args;
  },

  onLeave: function (log, retval, state) {
    log('  ----- out data:');
    log(hexdump(this.args[8], {length: this.args[9].toInt32()}));
  }
```

2 - Tracing the Network (HTTP/ S)

```bash

$ frida-trace -m "-[NSURLSession dataTaskWithRequest*]" -U <pid/name>

```

* edit scripts at __handlers__

```javascript
onEnter: function (log, args, state) {
    var request = new ObjC.Object(args[2]);
    log('-[NSURLSession dataTaskWithRequest:' + args[2] + ' completionHandler:' + args[3] + ']');
    log(' --- URL:');
    log(request.URL().toString());
    log(' --- headers:');
    log(request.allHTTPHeaderFields().toString());
    log(' --- body:');
    if (request.HTTPBody()) log(request.HTTPBody().toString());
    log(' --- method:');
    log(request.HTTPMethod().toString());
  }
```

* edit scripts at __handlers__

3 - Tracing Filesystem

```bash
frida-trace -m "-[NSFileManager fileExistsAtPath:]" -U TargetApp
```


```javascript
onEnter: function (args) {
    log('open' , ObjC.Object(args[2]).toString());
}
```

4 - Tracing with wildcards

```
frida-trace \
  -m "-[NSURLSession dataTaskWithRequest*]" \
  -U TargetApp
```

```javascript

// Wrap
var request = new ObjC.Object(args[2]);
// Access
request.toString();
request.URL().toString();
```

* at handlers 

```javascript

onEnter: function (log, args, state) {
  this.args=args;
},

onLeave: function (log, retval, state) {
  log(this.args[2].toInt32());
}
```

4-  Tracing Firebase

```bash

frida-trace -U -m "*[FIR* *collection*]" -m "*[FIR* *document*]" -Uf <app-bundle>
```

```javascript

//script_dump_firebase.js

var documentWithPath = ObjC.classes.FIRCollectionReference["- documentWithPath:"];
var collectionWithPath = ObjC.classes.FIRFirestore["- collectionWithPath:"];

Interceptor.attach(documentWithPath.implementation, {
    onEnter: function (args) {
        var message = ObjC.Object(args[2]);
        console.log("\n[FIRCollectionReference documentWithPath:@\"" + message.toString() + "\"]");
    }
});
Interceptor.attach(collectionWithPath.implementation, {
    onEnter: function (args) {
        var message = ObjC.Object(args[2]);
        console.log("\n[FIRFireStore collectionWithPath:@\"" + message.toString() + "\"]");
    }
});
```

```bash

frida --no-pause -U -l script_dump_firebase.js -Uf <package>

```


* if you got all data from firebase, use the script node bellow

```javascript
const firebase = require("firebase");
const util = require('util')

var firebaseConfig = {
    apiKey: "<redacted>",
    authDomain: "<redacted>",
    databaseURL: "<redacted>",
    projectId: "<redacted>",
    storageBucket: "<redacted>",
    messagingSenderId: "<redacted>",
    appId: "<redacted>",
};

firebase.initializeApp(firebaseConfig);
var database = firebase.firestore();

["<redacted>", "<redacted>"].forEach(collection => {
    database.collection(collection).get().then(function(querySnapshot) {
        console.log(`Data for ${collection}:`);
        querySnapshot.forEach((doc) => {
            var data = doc.data();
            console.log(util.inspect(data, false, null, true));
        });
    }).catch(function (error) {
        console.log("Error getting document:", error);
    });
});
```
