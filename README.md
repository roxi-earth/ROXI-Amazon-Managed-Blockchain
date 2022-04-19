# ROXI - Amazon Managed Blockchain



## Connection Via HTTP Provider

Example :

```javascript
import AWS from 'aws-sdk';
import HttpProvider from 'web3-providers-http';
import XHR2 from 'xhr2';

export default class AWSHttpProvider extends HttpProvider {
  send(payload, callback) {
    const self = this;
    const request = new XHR2(); // eslint-disable-line

    request.timeout = self.timeout;
    request.open('POST', self.host, true);
    request.setRequestHeader('Content-Type', 'application/json');

    request.onreadystatechange = () => {
      if (request.readyState === 4 && request.timeout !== 1) {
        let result = request.responseText; // eslint-disable-line
        let error = null; // eslint-disable-line

        try {
          result = JSON.parse(result);
        } catch (jsonError) {
          let message;
          if (!!result && !!result.error && !!result.error.message) {
            message = `[aws-ethjs-provider-http] ${result.error.message}`;
          } else  {
            message = `[aws-ethjs-provider-http] Invalid JSON RPC response from host provider ${self.host}: ` +
              `${JSON.stringify(result, null, 2)}`;
          }
          error = new Error(message);
        }

        callback(error, result);
      }
    };

    request.ontimeout = () => {
      callback(`[aws-ethjs-provider-http] CONNECTION TIMEOUT: http request timeout after ${self.timeout} ` +
        `ms. (i.e. your connect has timed out for whatever reason, check your provider).`, null);
    };

    try {
      const strPayload = JSON.stringify(payload);
      const region = process.env.AWS_DEFAULT_REGION || 'us-east-1';
      const credentials = new AWS.EnvironmentCredentials('AWS');
      const endpoint = new AWS.Endpoint(self.host);
      const req = new AWS.HttpRequest(endpoint, region);
      req.method = request._method;
      req.body = strPayload;
      req.headers['host'] = request._url.host;
      const signer = new AWS.Signers.V4(req, 'managedblockchain');
      signer.addAuthorization(credentials, new Date());
      request.setRequestHeader('Authorization', req.headers['Authorization']);
      request.setRequestHeader('X-Amz-Date', req.headers['X-Amz-Date']);
      request.send(strPayload);
    } catch (error) {
      callback(`[aws-ethjs-provider-http] CONNECTION ERROR: Couldn't connect to node '${self.host}': ` +
        `${JSON.stringify(error, null, 2)}`, null);
    }
  }
}

```



## Connection Via Websocket Provider

Example :

```javascript
import AWS from 'aws-sdk';
import WebsocketProvider  from 'web3-providers-ws';
import pkg from 'websocket';
const { w3cwebsocket } = pkg;
const Ws = w3cwebsocket;

export default class AWSWebsocketProvider extends WebsocketProvider {
  connect() {
    const region = process.env.AWS_DEFAULT_REGION || 'us-east-1';
    const credentials = new AWS.EnvironmentCredentials('AWS');
    const host = new URL(this.url).hostname;
    const endpoint = new AWS.Endpoint(`https://${host}/`);
    const req = new AWS.HttpRequest(endpoint, region);
    req.method = 'GET';
    req.body = '';
    req.headers['host'] = host;
    const signer = new AWS.Signers.V4(req, 'managedblockchain');
    signer.addAuthorization(credentials, new Date());
    const headers = {
      'Authorization': req.headers['Authorization'],
      'X-Amz-Date': req.headers['X-Amz-Date'],
      ...this.headers
    }
    this.connection = new Ws(this.url, this.protocol, undefined, headers, this.requestOptions, this.clientConfig);
    this._addSocketListeners();
  }
}
```



## Web3 Connection Via HTTP

Example :

```javascript
import Web3 from 'web3';
import AWSHttpProvider from './aws-http-provider.js';
const endpoint = process.env.AMB_HTTP_ENDPOINT
const web3 = new Web3(new AWSHttpProvider(endpoint));
web3.eth.getNodeInfo().then(console.log);
```



## Web3 Connection Via Websocket

Example :

```javascript
import Web3 from 'web3';
import AWSWebsocketProvider from "./aws-websocket-provider.js";
const endpoint = process.env.AMB_WS_ENDPOINT
const web3 = new Web3(new AWSWebsocketProvider(endpoint));
web3.eth.getNodeInfo().then(console.log).then(() => {
  web3.currentProvider.connection.close();
});
```

