# node-soap-example
A simple SOAP server and client example using the node-soap package and express.

# Introduction
I'm not an expert in Javascript, nor Node, nor SOAP, but I had the necessity to create a SOAP server using Node.js. Since I found the documentation to be a 
little poor, after the time I spent to make things somehow work, I want to share
what I did in case someone else needs this.

This README will go through a super simple yet kinda comprihensive example to
create a SOAP service and client in Node.js. The service has also been tested and is discoverable using .NET and its service reference tool.

# Example description
To show how things work, we will use a very simple example: we want to create a web service that allows a user to split a string message into its components, based on a user defined splitter character.

For instance, we want the service to split the string "id1:12:34:56:out42" using the character ":" as splitter (which should result into 'id1', '12', '34', '56', 'out42').

# Install
Install the required packages by running  `npm install` in the project 
directory, then you are ready to go.

Run the server with `npm start`, then test it with using the SOAP client with
`node client.js`.

# Code Explained
This section will explain the code, I hope this helps you to understand it better.

## The server
`app.js` contains the service code. Let's go through it.

We first define a function named `splitter_function`. This is the function our service will call in order to perform the split.

```
function splitter_function(args) {
    console.log('splitter_function');
    var splitter = args.splitter;
    var splitted_msg = args.message.split(splitter);
    var result = [];
    for(var i=0; i<splitted_msg.length; i++){
      result.push(splitted_msg[i]);
    }
    return {
        result: result
        }
}
```

This does nothing but reading the parameters and perform the split, pack the components into an array and return it. Please notice that there is absolutely zero parameter checking as it's not the scope of this example.

Next, we define the service, read the WSDL file and launch the server:
```
var serviceObject = {
  MessageSplitterService: {
        MessageSplitterServiceSoapPort: {
            MessageSplitter: splitter_function
        },
        MessageSplitterServiceSoap12Port: {
            MessageSplitter: splitter_function
        }
    }
};

// load the WSDL file
var xml = fs.readFileSync('service.wsdl', 'utf8');
// create express app
var app = express();

// root handler
app.get('/', function (req, res) {
  res.send('Node Soap Example!<br /><a href="https://github.com/macogala/node-soap-example#readme">Git README</a>');
})

var port = 8000;
// Launch the server and listen on *port*
app.listen(port, function () {
  console.log('Listening on port ' + port);
  var wsdl_path = "/wsdl";
  // create SOAP server that listens on *path*
  soap.listen(app, wsdl_path, serviceObject, xml);
  console.log("Check http://localhost:" + port + wsdl_path +"?wsdl to see if the service is working");
});
```

Please take note of the names we use in `serviceObject` as we will use them again.
In this example we use express to launch the server. Other frameworks may work too (eg. loopback).

Other information can be found [here](https://github.com/vpulim/node-soap#soaplistenserver-path-services-wsdl---create-a-new-soap-server-that-listens-on-path-and-provides-services).

## The WSDL
This section describes the WSDL file, the file that describes the web service. Some easy and quick information that I found useful can be found [here](https://www.tutorialspoint.com/wsdl/wsdl_introduction.htm).

A service can have multiple publicly available function. For semplicity, we will just define one, named `MessageSplitter`.

Since I had not tool that allowed me to auto generate the WSDL (maybe I'll spend sometime to
create one), I made the WSDL by hand. Please note that I am not an expert in WSDL or SOAP. I created the WSDL file starting from a WSDL file automatically generated from .NET, and used it as a reference to more or less understand how it works. Some parts are obscure to me, but hey, it works!

So, I'll try to explain the main parts of the WSDL file, without going too much in the details.

### WSDL types
A web service needs to define its inputs and outputs and how they are mapped into and out of the services. Inside the `<wsdl:types>` we define the data types the service will use. When we make a request to our function we will need to provide 2 input parameters, both strings. We define the request parameters into `<s:element name="MessageSplitterRequest">`, and specify the type, name and number of them using `<s:element>`. For example:

`<s:element minOccurs="1" maxOccurs="1" name="message" type="s:string"/>`

means that we have a parameter called message, that occurs exacly 1 time, and is of type `string`.

We then define the output structure creating `<s:element name="MessageSplitterResponse">`. Notice that the names are not important, we will map later what type need to be considered as input or output.

**IMPORTANT:** the parameters' names need to match those that will be used into the service and client functions (see app.js and client.js):

`app.js`
```
[...]
var splitter = args.splitter;
var splitted_msg = args.message.split(splitter);
[...]
```

### WSDL messages
Following the types, we have the messages part. `<wsdl:message>` describes the data being exchanged between the service and the client. We define two messages, one for the input `MessageSplitterMessageIn`, that refers to the request data type, and one for the output, `MessageSplitterMessageOut`, that refers to the response data type.

```
<wsdl:message name="MessageSplitterMessageIn">
  <wsdl:part name="parameters" element="tns:MessageSplitterRequest"/>
</wsdl:message>
<wsdl:message name="MessageSplitterMessageOut">
  <wsdl:part name="parameters" element="tns:MessageSplitterResponse"/>
</wsdl:message>
```

### WSDL portType
We then have the portType, which combines multiple elements to form a complete one-way or round-trip operation.

We define one single portType `<wsdl:portType name="MessageSplitterPort">`, that defines one operation which consists of an input and output message, defined above in the WSDL message section.
```
<wsdl:operation name="MessageSplitter">
  <wsdl:input message="tns:MessageSplitterMessageIn"/>
  <wsdl:output message="tns:MessageSplitterMessageOut"/>
</wsdl:operation>
```

**IMPORTANT:** the `<wsdl:operation>` name has to match the one specified into the service object in the SOAP service definition in `app.js`:
```
[...]
MessageSplitterServiceSoapPort: {
  MessageSplitter: splitter_function
},
[...]
```

### WSDL Binding
Bindings provide details on how a portType operation will actually be transmitted over the wire. There are 2 extremely similar bindings, check them out: they only differ from the version.

We create a binding with:
```
<wsdl:binding name="MessageSplitterServiceSoap" type="tns:MessageSplitterPort">
```
the `name` defines the name of the binding and is not so important, it will referred later. The `type` refers to the port for the binding, and refers to the port we previously defined.

Inside the `<wsdl:binding>`, we define a `<wsdl:operation>`:
```
<wsdl:operation name="MessageSplitter">
<soap:operation soapAction="http://tempuri.org/MessageSplitter" style="document"/>
```
the `name` and the `soapAction` have to be the same as the name specified in the `<wsdl:operation>` inside the `<wsdl:portType>`, or apparently it does not work (don't know why, I told you there are still a lot of obscure things).

There 2 very similar bindings in the example.

### WSDL Service
Last but not least, the `<wsdl:service>` defines the port supported by the web service. For each supported protocol, there is one port element.
From the service element, clients can learn:
1. where to access the service
2. which port to use to access the service
3. how the communication messages are defined

Each `<wsdl:port>` has 2 elements: a `name` and a `binding`. The `name` has to be unique among all the ports, the `binding` must refer to a binding previously defined.

Now, recall the service definition in the `app.js` file:
```
var serviceObject = {
  MessageSplitterService: {
        MessageSplitterServiceSoapPort: {
            MessageSplitter: splitter_function
        },
        MessageSplitterServiceSoap12Port: {
            MessageSplitter: splitter_function
        }
    }
};
```
The `<wsdl:service name>` and the port `<wsdl:port name>` must match the names specified in `app.js`.

## Client
The `client.js` file specifies a client.
```
var soap = require('soap');
var url = 'http://localhost:8000/wsdl?wsdl';

// Create client
soap.createClient(url, function (err, client) {
  if (err){
    throw err;
  }
  var args = {
    message: "id1:12:34:56:out42",
    splitter: ":"
  };
  // call the service
  client.MessageSplitter(args, function (err, res) {
    if (err)
      throw err;
      // print the service returned result
    console.log(res); 
  });
});
```
The file is self-explainatory, just make sure you create the args with the same parameters names specified in the wsdl file.

# References
1. [https://www.tutorialspoint.com/wsdl](https://www.tutorialspoint.com/wsdl
2. [https://github.com/vpulim/node-soap](https://github.com/vpulim/node-soap)
3. [https://github.com/ezzye/nodejs_mock_agresso](https://github.com/ezzye/nodejs_mock_agresso)