# Alexa code snippets

This repository contains some sample of the skills which can be used as snippets in applications. 

It is **work in progress** at the moment. 

### Current snippets (2): 

#### How to call a external service

Most of the times you will need to call an external service to handle the request. The below code snippet show how can you call an external service. 

Create a login function to login. Here is an example of Salesforce (File name: sfdc-handler):
```
const SF_API_URL = '/services/apexrest/api/<YOUR ENDPOINT>';
const SF_LOGIN_URL = 'https://test.salesforce.com/services/oauth2/token';
const UTF8_ENCODING = "utf-8";


let sfdc_login = function () {
    let requestInstance = request('POST', SF_LOGIN_URL, {
        qs: {
            grant_type: "password",
            username: process.env.USERNAME,
            password: process.env.PASSWORD + process.env.SECURITY_TOKEN,
            client_id: process.env.CLIENT_ID,
            client_secret: process.env.CLIENT_SECRET
        }
    });

    let response = JSON.parse(requestInstance.getBody(UTF8_ENCODING));
    return {
        'access_token': response.access_token,
        'instance_url': response.instance_url
    };
};
```

Actual service call (File name: sfdc-handler):

```
module.exports.bookAppointment = function () {
    let login = sfdc_login();
    let body = {
       // Construct your request here
    };
    // Request is a NODE JS request module available on NPM
    let res = request('POST', login.instance_url + SF_API_URL, {
        headers: {
            Authorization: "Bearer " + login.access_token
        },
        json: {
            // Add body here for post request
        }
    });
    return parseResponse(res);
};
```

Call this service from a handler (File name: index.js)

```
// Top level import
const sfdc = require("./sfdc-handler");


const CompleteBookAppointmentIntent = {
    canHandle(handlerInput) {
        const request = handlerInput.requestEnvelope.request;

        return request.type === 'IntentRequest'
            && request.intent.name === 'YOUR INTENT NAME'
            && request.dialogState === 'COMPLETED';
    },
    async handle(handlerInput) {
        const currentIntent = handlerInput.requestEnvelope.request.intent;
        let sfdc_response = sfdc.bookAppointment();
        if (sfdc_response && sfdc_response.message === 'DONE') {
            return handlerInput.responseBuilder
                .speak('Completed')
                .reprompt('Completed')
                .getResponse();
        } else {
            return handlerInput.responseBuilder
                .speak('Sorry')
                .reprompt('Some error')
                .getResponse();
        }
    }
};
```
#### How to play a message while waiting for the response. 
It take some time for service to respond and the default behaviour of Alexa is to just wait silently for 8 seconds. You can notify user that the request is processing. 

```
function callDirectiveService(handlerInput) {
    // Call Alexa Directive Service.
    const requestEnvelope = handlerInput.requestEnvelope;
    const directiveServiceClient = handlerInput.serviceClientFactory.getDirectiveServiceClient();

    const requestId = requestEnvelope.request.requestId;
    const endpoint = requestEnvelope.context.System.apiEndpoint;
    const token = requestEnvelope.context.System.apiAccessToken;
    
    // build the progressive response directive
    const directive = {
        header: {
            requestId,
        },
        directive: {
            type: "VoicePlayer.Speak",
            speech: "Confirming your appointment. Please wait..."
        },
    };
    // send directive
    return directiveServiceClient.enqueue(directive, endpoint, token);
}

```

Once the above function is created, call this function inside your webservice as 

```
// Top level import
const sfdc = require("./sfdc-handler");


const CompleteBookAppointmentIntent = {
    canHandle(handlerInput) {
        const request = handlerInput.requestEnvelope.request;

        return request.type === 'IntentRequest'
            && request.intent.name === 'YOUR INTENT NAME'
            && request.dialogState === 'COMPLETED';
    },
    async handle(handlerInput) {
        //*************************************************
        await callDirectiveService(handlerInput, 'MODIFY');
        //*************************************************
        const currentIntent = handlerInput.requestEnvelope.request.intent;
        let sfdc_response = sfdc.bookAppointment();
        if (sfdc_response && sfdc_response.message === 'DONE') {
            return handlerInput.responseBuilder
                .speak('Completed')
                .reprompt('Completed')
                .getResponse();
        } else {
            return handlerInput.responseBuilder
                .speak('Sorry')
                .reprompt('Some error')
                .getResponse();
        }
    }
};
```