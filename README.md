# AWS SQS Consumer

**Description**:  Polls the Amazon Web Services Simple Queue Service and dispatches messages to the listeners.
Messages handy functions, such as `delete` or `changeVisibility`, and the body is transformed by a transformer
function.

## Dependencies

This package was meant to be used along with Typescript. The only production dependency is the AWS SDK. 

## Installation

`npm i @bausano/sqs-consumer`

## Configuration

Since this package is built on top of the AWS SDK, the correct access tokens and regions have to be
set in the node enviroment variables.
Please refer to [this guide](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/configuring-the-jssdk.html)
for further instructions on how to configure the service.

When constructing the `SQS` object for consumer, lock the version of the APIS:

`const sqs: AWS.SQS = new AWS.SQS({ apiVersion: '2012-11-05' })`

And set the correct region either in env variables or in your codebase:

`AWS.config.update({ region: '...' })`

## Usage

The consumer emits `QueueMessage` instances to listeners.
Its body has been transformed via provided transform function
into type `T`.

### Transformer
For example, if your messages carry user information, you
can do following:

```
export default (body: string) : User => {
    const { name, email } : any = JSON.parse(body)

    return new User(name, email)
}
```

Should there be an exception thrown during the transform function,
an error is emitted to error listeners and messages is left
in queue.

It could be useful to transform the body into an object. You can use
`any` type or, preferably, create an interface and export the interface.

```
// Action.ts
export interface Action {
    name: string
    source: number
    target?: number
}

// transformer.ts
export default (body: string) : Action {
    const { name, source, target } : any = JSON.parse(body)

    // Ensure you have appropriate max receive count option in your SQS
    // if you want to throw errors in transformer as it does not delete
    // messages that fail transformation.
    if (name === undefined || source === undefined) {
        throw new Error('Message body missing necessary parameters.')
    }

    return { name, source, target }
}

// main.ts
// Your app would be initialized like so:
const app: QueueConsumer<Action> = new QueueConsumer(sqs, config, transformer)
```

### Config
The config this consumer requires extends `AWS.SQS.Types.ReceiveMessageRequest`
interface. Documentation can be found [here](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/SQS.html#receiveMessage-property) in the
**Parameters** section.
You can find it also in the AWS Github repo [here](https://github.com/aws/aws-sdk-js/blob/master/apis/sqs-2012-11-05.normal.json) or [here](https://github.com/aws/aws-sdk-js/blob/master/clients/sqs.d.ts) (search for `ReceiveMessageRequest`).

On top of these parameters, this library adds `Interval?: number`. This has to be set for continious polling.

### Listeners
There are two groups of listeners you can make use of: `QueueMessage`, `ConsumerException`. To add listeners to the app,
you have to init new instance of the consumer and use following API:

`app.onMessage.addListener(message => handler(message))`

Where message is of type `QueueMessage` and has property `body` of type that you specified on
init (for example mentioned above, it would `body: Action`).

To listen to errors, you add a listener `(error: ConsumerException) => void`:

`app.onError.addListener(error => handler(error))`

There are 3 types of error reported, all of which `extends ConsumerException`:
- from connecting to SQS, corresponds to `class ConnectionException`
- transforming messages, corresponds to `class TransformerException`
- handling messages, corresponds to `class ListenerException`

On `class ConsumerException`, there is one public method: `unwrap () : Error`.
This gives you an instance of `Error` that is responsible for the exception.

### Example
```
/**
 * Creates new sqs consumer with configuration that
 * is just an extended AWS.SQS.Types.ReceiveMessageRequest object
 * and tranform function that assigns type of T as message body.
 *
 * @var {QueueConsumer<T>}
 */

const app: QueueConsumer<T> = new QueueConsumer(
  new AWS.SQS(),
  config,
  transform
)

/**
 * Message handler of type
 * (message: QueueMessage<T>) => void
 */

app.onMessage.addListener(m => flow(m))

/**
 * Error handlers of type
 * (error: ConsumerException) => void
 */

app.onError
  .addListener(console.log)
  .addListener(e => publish(e))

/**
 * Starts the queue consumer.
 */

app.run()

// or app.runOnce() for AWS Lambda services.

/**
 * Stops the polling.
 */

app.stop()
```

### QueueMessage
`QueueMessage` has following methods and properties:

- `body: T` is transformed message body
- `receipt: string` is the SQS message receipt
- `raw: AWS.SQS.Message` is the raw SQS message from the SDK package
- `changeVisibility (secs: number) : Promise<AWS.Respose>` changes the message visibility
- `delete () : Promise<AWS.Respose>` removes the message

This library is trying to work with AWS SDK as closely as possible. To use it,
you can often refer to the official documentation, as under the hood these methods often are just
`return sqs.method(request).promise()`.

----

## Open source licensing info

1. [LICENSE](LICENSE)
2. [CFPB Source Code Policy](https://github.com/cfpb/source-code-policy/)


----

## Credits and references

This library is inpired by [bbc/sqs-consumer](https://github.com/bbc/sqs-consumer) project.
