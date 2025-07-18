---
navigation_title: Webhook action
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/actions-webhook.html
applies_to:
  stack: ga
products:
  - id: elasticsearch
---

# Webhook action [actions-webhook]

Use the `webhook` action to send a request to any web service. The webhook action supports both HTTP and HTTPS connections. See [Webhook action attributes](#webhook-action-attributes) for the supported attributes.

## Configuring webhook actions [configuring-webook-actions]

You configure webhook actions in the `actions` array. Action-specific attributes are specified using the `webhook` keyword.

The following snippet shows a simple webhook action definition:

```js
"actions" : {
  "my_webhook" : { <1>
    "transform" : { ... }, <2>
    "throttle_period" : "5m", <3>
    "webhook" : {
      "method" : "POST", <4>
      "host" : "mylisteningserver", <5>
      "port" : 9200, <6>
      "path": "/{{ctx.watch_id}}", <7>
      "body" : "{{ctx.watch_id}}:{{ctx.payload.hits.total}}" <8>
    }
  }
}
```

1. The id of the action
2. An optional [transform](transform.md) to transform the payload before executing the `webhook` action
3. An optional [throttle period](actions.md#actions-ack-throttle) for the action (5 minutes in this example)
4. The HTTP method to use when connecting to the host
5. The host to connect to
6. The port to connect to
7. The path (URI) to use in the HTTP request
8. The body to send with the request

You can use basic authentication when sending a request to a secured webservice. For example, the following `webhook` action creates a new issue in GitHub:

```js
"actions" : {
  "create_github_issue" : {
    "transform": {
      "script": "return ['title':'Found errors in \\'contact.html\\'', 'body' : 'Found ' + ctx.payload.hits.total + ' errors in the last 5 minutes', 'assignee' : 'web-admin', 'labels' : ['bug','sev2']]"
    },
    "webhook" : {
      "method" : "POST",
      "url" : "https://api.github.com/repos/<owner>/<repo>/issues",
      "body": "{{#toJson}}ctx.payload{{/toJson}}",
      "auth" : {
        "basic" : {
          "username" : "<username>", <1>
          "password" : "<password>"
        }
      }
    }
  }
}
```

1. The username and password for the user creating the issue

::::{note}
By default, both the username and the password are stored in the `.watches` index in plain text. When the {{es}} {{security-features}} are enabled, {{watcher}} can encrypt the password before storing it.
::::

You can also use PKI-based authentication when submitting requests to a cluster that has {{es}} {{security-features}} enabled. When you use PKI-based authentication instead of HTTP basic auth, you don’t need to store any authentication information in the watch itself. To use PKI-based authentication, you [configure the SSL key settings](elasticsearch://reference/elasticsearch/configuration-reference/watcher-settings.md#ssl-notification-settings) for {{watcher}} in [`elasticsearch.yml`](/deploy-manage/stack-settings.md).

## Query parameters [webhook-query-parameters]

You can specify query parameters to send with the request with the `params` field. This field simply holds an object where the keys serve as the parameter names and the values serve as the parameter values:

```js
"actions" : {
  "my_webhook" : {
    "webhook" : {
      "method" : "POST",
      "host" : "mylisteningserver",
      "port" : 9200,
      "path": "/alert",
      "params" : {
        "watch_id" : "{{ctx.watch_id}}" <1>
      }
    }
  }
}
```

1. The parameter values can contain templated strings.

## Custom request headers [webhook-custom-request-headers]

You can specify request headers to send with the request with the `headers` field. This field simply holds an object where the keys serve as the header names and the values serve as the header values:

```js
"actions" : {
  "my_webhook" : {
    "webhook" : {
      "method" : "POST",
      "host" : "mylisteningserver",
      "port" : 9200,
      "path": "/alert/{{ctx.watch_id}}",
      "headers" : {
        "Content-Type" : "application/yaml" <1>
      },
      "body" : "count: {{ctx.payload.hits.total}}"
    }
  }
}
```

1. The header values can contain templated strings.

## Webhook action attributes [_webhook_action_attributes]

$$$webhook-action-attributes$$$

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `scheme` | no | http | The connection scheme. Valid values are: `http` or `https`. |
| `host` | yes | - | The host to connect to. |
| `port` | yes | - | The port the HTTP service is listening on. |
| `path` | no | - | The URL path. The path can be static text or include Mustache                                                    [templates](how-watcher-works.md#templates). URL query string parameters must be                                                    specified via the `request.params` attribute. |
| `method` | no | get | The HTTP method. Valid values are: `head`, `get`, `post`, `put`                                                    and `delete`. |
| `headers` | no | - | The HTTP request headers. The header values can be static text                                                    or include Mustache [templates](how-watcher-works.md#templates). |
| `params` | no | - | The URL query string parameters. The parameter values can be                                                    static text or include Mustache [templates](how-watcher-works.md#templates). |
| `auth` | no | - | Authentication related HTTP headers. Currently, only basic                                                    authentication is supported. |
| `body` | no | - | The HTTP request body. The body can be static text or include                                                    Mustache [templates](how-watcher-works.md#templates). When not specified, an empty                                                    body is sent. |
| `proxy.host` | no | - | The proxy host to use when connecting to the host. |
| `proxy.port` | no | - | The proxy port to use when connecting to the host. |
| `connection_timeout` | no | 10s | The timeout for setting up the http connection. If the connection                                                    could not be set up within this time, the action will timeout and                                                    fail. |
| `read_timeout` | no | 10s | The timeout for reading data from http connection. If no response                                                    was received within this time, the action will timeout and fail. |
| `url` | no | - | A shortcut for specifying the request scheme, host, port, and                                                    path as a single string. For example, `http://example.org/foo/my-service`. |
