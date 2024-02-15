# Call for News GPT: Generative AI Phone Calling

Wouldn't it be neat if you could build an app that allowed you to chat with ChatGPT on the phone?

This app serves as a demo exploring Cohere/OpenAI:
- [OpenAI](https://openai.com) for GPT prompt completion

## Setting up for Development

### Prerequisites
Sign up for the following services and get an API key for each:
- [OpenAI](https://platform.openai.com/signup)


If you're hosting the app locally, we also recommend using a tunneling service like [ngrok](https://ngrok.com) so that Twilio can forward audio to your app.


### 1. Start Ngrok
Start an [ngrok](https://ngrok.com) tunnel for port `3000`:

```bash
ngrok http 3000
```
Ngrok will give you a unique URL, like `abc123.ngrok.io`. Copy the URL without http:// or https://. You'll need this URL in the next step.

### 2. Configure Environment Variables
Copy `.env.example` to `.env` and configure the following environment variables:

```bash
# Your ngrok or server URL
# E.g. 123.ngrok.io or myserver.fly.dev
SERVER="yourserverdomain.com"

# Service API Keys
OPENAI_API_KEY="sk-XXXXXX"

# Configure your Twilio credentials if you want
# to make test calls using '$ npm test'.
TWILIO_ACCOUNT_SID="YOUR-ACCOUNT-SID"
TWILIO_AUTH_TOKEN="YOUR-AUTH-TOKEN"
FROM_NUMBER='+12223334444'
TO_NUMBER='+13334445555'
```

### 3. Install Dependencies with NPM
Install the necessary packages:

```bash
npm install
```

### 4. Start Your Server in Development Mode
Run the following command:
```bash
npm run dev
```
This will start your app using `nodemon` so that any changes to your code automatically refreshes and restarts the server.

### 5. Configure an Incoming Phone Number

Connect a phone number using the [Twilio Console](https://console.twilio.com/us1/develop/phone-numbers/manage/incoming).

You can also use the Twilio CLI:

```bash
twilio phone-numbers:update +1[your-twilio-number] --voice-url=https://your-server.ngrok.io/incoming
```
This configuration tells Twilio to send incoming call audio to your app when someone calls your number. The app responds to the incoming call webhook with a [Stream](https://www.twilio.com/docs/voice/twiml/stream) TwiML verb that will connect an audio media stream to your websocket server.

## Application Workflow
CallGPT coordinates the data flow between multiple different services:
![Call GPT Flow]


## Modifying the ChatGPT Context & Prompt
Within `gpt-service.js` you'll find the settings for the GPT's initial context and prompt. For example:

```javascript
this.userContext = [
  { "role": "system", "content": "You are an news reader. You have a youthful and cheery personality.Add a '•' symbol every 5 to 10 words at natural pauses where your response can be split for text to speech." },
  { "role": "assistant", "content": "Hello! I understand you're looking for news, is that correct?" },
],
```
### About the `system` Attribute
The `system` attribute is background information for the GPT. As you build your use-case, play around with modifying the context. A good starting point would be to imagine training a new employee on their first day and giving them the basics of how to help a customer.

These context items help shape a GPT so that it will act more naturally in a phone conversation.

The `•` symbol context in particular is helpful for the app to be able to break sentences into natural chunks. This speeds up text-to-speech processing so that users hear audio faster.

### About the `content` Attribute
This attribute is your default conversations starter for the GPT. However, you could consider making it more complex and customized based on personalized user data.

In this case, our bot will start off by saying, "Hello! I understand you called for news, is that correct?"

2. When we call GPT for completions, we also pass in the same `function-manifest` JSON as the tools parameter. This allows the GPT to "know" what functions are available:

```javascript
const stream = await this.openai.chat.completions.create({
  model: 'gpt-4',
  messages: this.userContext,
  tools, // <-- function-manifest definition
  stream: true,
});
```
3. When the GPT responds, it will send us a stream of chunks for the text completion. The GPT will tell us whether each text chunk is something to say to the user, or if it's a tool call that our app needs to execute.  This is indicated by the `deltas.tool_calls` key:
```javascript
if (deltas.tool_calls) {
  // handle function calling
}
```
4. Once we have gathered all of the stream chunks about the tool call, our application can run the actual function code that we imported during the first step. The function name and parameters are provided by GPT:
```javascript
const functionToCall = availableFunctions[functionName];
const functionResponse = functionToCall(functionArgs);
```
5. As the final step, we add the function response data into the conversation context like this:

```javascript
this.userContext.push({
  role: 'function',
  name: functionName,
  content: functionResponse,
});
```
We then ask the GPT to generate another completion including what it knows from the function call. This allows the GPT to respond to the user with details gathered from the external data source.

---------
### Adding Custom Function Calls
You can have your GPT call external data sources by adding functions to the `/functions` directory. 

### Returning Arguments to GPT
Your function should always return a value: GPT tends to get confused when the function returns nothing, and may continue trying to call the function expecting an answer. If your function doesn't have any data to return to the GPT, you should still consider returning a response that says something like "The function to do (X) ran successfully."

Any data that you return to the GPT should match the expected format listed in the `returns` key of `function-manifest.js`.

## Utility Scripts for Placing Calls
The `scripts` directory contains two files that allow you to place test calls:
- `npm run inbound` will place an automated call from a Twilio number to your app and speak a script. You can adjust this to your use-case, e.g. as an automated test.
- `npm run outbound` will place an outbound call that connects to your app. This can be useful if you want the app to call your phone so that you can manually test it.

## Testing with Jest
Repeatedly calling the app can be a time consuming way to test your tool function calls. This project contains example unit tests that can help you test your functions without relying on the GPT to call them.

Simple example tests are available in the `/test` directory. To run them, simply run `npm run test`.

## Deploy via Fly.io
Fly.io is a hosting service similar to Heroku that simplifies the deployment process. Given Twilio Media Streams are sent and received from us-east-1, it's recommended to choose Fly's Ashburn, VA (IAD) region.

> Deploying to Fly.io is not required to try the app, but can be helpful if your home internet speed is variable.

Modify the app name `fly.toml` to be a unique value (this must be globally unique).

Deploy the app using the Fly.io CLI:
```bash
fly launch

fly deploy
```

Import your secrets from your .env file to your deployed app:
```bash
fly secrets import < .env
```
