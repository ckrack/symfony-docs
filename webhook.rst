Webhook
=======

.. versionadded:: 6.3

    The Webhook component was introduced in Symfony 6.3.

Webhooks are event notification mechanisms typically delivered via HTTP POST requests, enabling real-time updates between systems.

The Webhook component provides two primary capabilities:

1. **Consuming**: Receive and process webhook calls from remote systems
2. **Sending**: Dispatch webhooks to registered endpoints when events occur

This document covers both capabilities in the context of a full-stack Symfony application.

Installation
------------

.. code-block:: terminal

    $ composer require symfony/webhook


Consuming Webhooks
------------------

The Webhook component, combined with RemoteEvent, enables you to receive and process webhooks through three key phases:

1. Receiving the webhook via a dedicated endpoint
2. Verifying the webhook and converting it to a RemoteEvent object
3. Consuming the event in your application logic

A Centralized Webhook Endpoint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The :class:`Symfony\\Component\\Webhook\\Controller\\WebhookController` provides a single entry point for receiving all incoming webhooks, regardless of their source (third-party services, custom APIs, etc.).

By default, any URL prefixed with ``/webhook`` routes to this controller. You can customize this prefix in your routing configuration:

.. configuration-block::

    .. code-block:: yaml

        # config/routes/webhook.yaml
        webhook:
            resource: '@FrameworkBundle/Resources/config/routing/webhook.xml'
            prefix: /webhook  # customize as needed

    .. code-block:: xml

        <!-- config/routes/webhook.xml -->
        <routes xmlns="http://symfony.com/schema/routing"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/routing
                https://symfony.com/schema/routing/routing-1.0.xsd">
            <import resource="@FrameworkBundle/Resources/config/routing/webhook.xml"
                prefix="/webhook" />
        </routes>

    .. code-block:: php

        // config/routes/webhook.php
        use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;

        return static function (RoutingConfigurator $routes): void {
            $routes->import('@FrameworkBundle/Resources/config/routing/webhook.xml')
                ->prefix('/webhook');
        };

Next, configure the parser services that will handle incoming webhooks.
The controller uses a routing mechanism to map incoming requests to the
appropriate parser:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/webhook.yaml
        framework:
            webhook:
                routing:
                    acme_webhook:  # routing name, maps to /webhook/acme_webhook
                        service: App\Webhook\AcmeWebhookRequestParser
                        secret: '%env(WEBHOOK_SECRET)%'  # optional

    .. code-block:: xml

        <!-- config/packages/framework.xml -->
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:framework="http://symfony.com/schema/dic/symfony"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/symfony
                https://symfony.com/schema/dic/symfony/symfony-1.0.xsd">
            <framework:config>
                <framework:webhook enabled="true">
                    <framework:routing type="acme_webhook">
                        <framework:service>App\Webhook\AcmeWebhookRequestParser</framework:service>
                        <framework:secret>%env(WEBHOOK_SECRET)%</framework:secret>
                    </framework:routing>
                </framework:webhook>
            </framework:config>
        </container>

    .. code-block:: php

        // config/packages/framework.php
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $config): void {
            $config->webhook()
                ->routing('acme_webhook')
                ->service('App\Webhook\AcmeWebhookRequestParser')
                ->secret('%env(WEBHOOK_SECRET)%');
        };

The routing name becomes part of the webhook URL (e.g.,
``https://example.com/webhook/acme_webhook``). Each routing name must be
unique and connects the webhook source to your consumer code.

All parsers are automatically injected into the WebhookController.

Parsing Webhook Requests
~~~~~~~~~~~~~~~~~~~~~~~~

Once a webhook request arrives at your endpoint, it must be parsed and validated before
your application can process it. Parsing involves verifying the request's authenticity
(typically via signature validation), extracting the payload, and converting it into a
:class:`Symfony\\Component\\RemoteEvent\\RemoteEvent` object that your application can work with.

Symfony provides two approaches to handle parsing:

1. **Built-in Parser**: Use the standard :class:`Symfony\\Component\\Webhook\\Client\\RequestParser` for webhooks from other Symfony applications
2. **Custom Parser**: Create your own parser for webhooks from third-party services or custom APIs

Choose the approach that best fits your webhook source.

Using the Built-in Parser
^^^^^^^^^^^^^^^^^^^^^^^^^

For webhooks originating from other Symfony applications, you can use the built-in
:class:`Symfony\\Component\\Webhook\\Client\\RequestParser` instead of creating a custom parser.

This parser handles the standard Symfony webhook request format and is perfect for
consuming webhooks from Symfony apps:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/framework.yaml
        framework:
            webhook:
                routing:
                    acme_webhook:
                        service: Symfony\Component\Webhook\Client\RequestParser
                        secret: '%env(WEBHOOK_SECRET)%'

    .. code-block:: xml

        <!-- config/packages/framework.xml -->
        <framework:config>
            <framework:webhook enabled="true">
                <framework:routing type="acme_webhook">
                    <framework:service>Symfony\Component\Webhook\Client\RequestParser</framework:service>
                    <framework:secret>%env(WEBHOOK_SECRET)%</framework:secret>
                </framework:routing>
            </framework:webhook>
        </framework:config>

    .. code-block:: php

        // config/packages/framework.php
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $config): void {
            $config->webhook()
                ->routing('acme_webhook')
                ->service(Symfony\Component\Webhook\Client\RequestParser::class)
                ->secret('%env(WEBHOOK_SECRET)%');
        };

The built-in parser automatically handles request validation and signature verification,
allowing you to focus on consuming the RemoteEvent in your application logic.

Creating a Custom Parser
^^^^^^^^^^^^^^^^^^^^^^^^^

For webhooks from custom APIs, implement a parser using the :class:`Symfony\\Component\\Webhook\\Client\\RequestParserInterface` or extend :class:`Symfony\\Component\\Webhook\\Client\\AbstractRequestParser`.

The easiest way is using the maker command:

.. code-block:: terminal

    $ php bin/console make:webhook

.. tip::

    Starting in `MakerBundle`_ ``v1.58.0``, the ``make:webhook`` command generates both the parser and consumer classes, and updates your configuration automatically.

When extending :class:`Symfony\\Component\\Webhook\\Client\\AbstractRequestParser`, you need to implement two methods:

:method:`Symfony\\Component\\Webhook\\Client\\AbstractRequestParser::getRequestMatcher` - Validate the incoming request format:

.. code-block:: php

    // src/Webhook/AcmeWebhookRequestParser.php
    namespace App\Webhook;

    use Symfony\Component\Webhook\Client\AbstractRequestParser;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\RequestMatcher\ChainRequestMatcher;
    use Symfony\Component\HttpFoundation\RequestMatcher\IsJsonRequestMatcher;
    use Symfony\Component\HttpFoundation\RequestMatcher\MethodRequestMatcher;
    use Symfony\Component\HttpFoundation\RequestMatcher\RequestMatcherInterface;
    use Symfony\Component\RemoteEvent\RemoteEvent;

    final class AcmeWebhookRequestParser extends AbstractRequestParser
    {
        protected function getRequestMatcher(): RequestMatcherInterface
        {
            return new ChainRequestMatcher([
                new IsJsonRequestMatcher(),
                new MethodRequestMatcher('POST'),
            ]);
        }

        protected function doParse(
            Request $request,
            #[\SensitiveParameter] string $secret
        ): ?RemoteEvent {
            $payload = json_decode(
                $request->getContent(),
                true,
                flags: JSON_THROW_ON_ERROR
            );

            return new RemoteEvent(
                $payload['event_type'],
                $payload['event_id'],
                $payload,
            );
        }
    }

:method:`Symfony\\Component\\Webhook\\Client\\AbstractRequestParser::doParse`
- Verify the webhook and parse it into a RemoteEvent:

The method receives the request and secret. You should:

- Validate the request signature (typically HMAC-SHA256)
- Parse and validate the payload
- Throw :class:`Symfony\\Component\\Webhook\\Exception\\RejectWebhookException` for invalid requests
- Return a :class:`Symfony\\Component\\RemoteEvent\\RemoteEvent` on success

Handling Complex Payload Transformations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For complex webhook payloads, use the
:class:`Symfony\\Component\\RemoteEvent\\PayloadConverterInterface` to
encapsulate transformation logic:

.. code-block:: php

    // src/RemoteEvent/AcmeWebhookPayloadConverter.php
    namespace App\RemoteEvent;

    use Symfony\Component\RemoteEvent\PayloadConverterInterface;
    use Symfony\Component\RemoteEvent\RemoteEvent;

    final class AcmeWebhookPayloadConverter
        implements PayloadConverterInterface
    {
        public function convert(array $payload): RemoteEvent
        {
            // Map external event names to your domain events
            $eventName = match ($payload['event_type']) {
                'resource.created' => 'acme.resource_created',
                'resource.updated' => 'acme.resource_updated',
                'resource.deleted' => 'acme.resource_deleted',
                default => 'acme.unknown_event',
            };

            return new RemoteEvent(
                $eventName,
                $payload['event_id'],
                $payload
            );
        }
    }

Then use it in your parser:

.. code-block:: php

    use Symfony\Component\Webhook\Exception\RejectWebhookException;

    protected function doParse(
        Request $request,
        #[\SensitiveParameter] string $secret
    ): ?RemoteEvent {
        try {
            $payload = json_decode(
                $request->getContent(),
                true,
                flags: JSON_THROW_ON_ERROR
            );

            return $this->converter->convert($payload);
        } catch (ParseException|JsonException $e) {
            throw new RejectWebhookException(
                406,
                $e->getMessage(),
                $e
            );
        }
    }

Consuming the RemoteEvent
~~~~~~~~~~~~~~~~~~~~~~~~~

Whether processed synchronously or asynchronously (via Messenger), you
need a consumer implementing
:class:`Symfony\\Component\\RemoteEvent\\Consumer\\ConsumerInterface`.

The ``make:webhook`` command generates one automatically. Otherwise, create
it manually using the
:class:`Symfony\\Component\\RemoteEvent\\Attribute\\AsRemoteEventConsumer`
attribute:

.. code-block:: php

    // src/RemoteEvent/AcmeWebhookConsumer.php
    namespace App\RemoteEvent;

    use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
    use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
    use Symfony\Component\RemoteEvent\RemoteEvent;

    #[AsRemoteEventConsumer('acme_webhook')]  // must match routing name
    final class AcmeWebhookConsumer implements ConsumerInterface
    {
        public function consume(RemoteEvent $event): void
        {
            // Handle the event based on your business logic
        }
    }

The routing name in the attribute must match the configuration entry.

Asynchronous Processing
^^^^^^^^^^^^^^^^^^^^^^^

By default, webhook consumers are invoked synchronously when the
RemoteEvent is dispatched. To process webhooks asynchronously using
Messenger, configure routing for the
:class:`Symfony\\Component\\RemoteEvent\\Messenger\\ConsumeRemoteEventMessage`:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/messenger.yaml
        framework:
            messenger:
                routing:
                    'Symfony\Component\RemoteEvent\Messenger\ConsumeRemoteEventMessage': async

    .. code-block:: xml

        <!-- config/packages/messenger.xml -->
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:framework="http://symfony.com/schema/dic/symfony"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/symfony
                https://symfony.com/schema/dic/symfony/symfony-1.0.xsd">
            <framework:config>
                <framework:messenger>
                    <framework:routing
                        message-class="Symfony\Component\RemoteEvent\Messenger\ConsumeRemoteEventMessage">
                        <framework:bus>async</framework:bus>
                    </framework:routing>
                </framework:messenger>
            </framework:config>
        </container>

    .. code-block:: php

        // config/packages/messenger.php
        use Symfony\Component\RemoteEvent\Messenger\ConsumeRemoteEventMessage;
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $config): void {
            $config->messenger()
                ->routing(ConsumeRemoteEventMessage::class)
                ->bus('async');
        };

With this configuration, consumers are invoked asynchronously via the
message bus. Without it, consumers are processed synchronously during
the webhook handling.


Built-in Integrations
~~~~~~~~~~~~~~~~~~~~~

Symfony provides pre-built parsers for common services, so you don't need to create custom parsers for them.

Mailer Webhooks
^^^^^^^^^^^^^^^

Receive delivery and engagement notifications from third-party mailers:

============== ==========================================
Mailer Service Parser service name
============== ==========================================
Brevo          ``mailer.webhook.request_parser.brevo``
Mailgun        ``mailer.webhook.request_parser.mailgun``
Mailjet        ``mailer.webhook.request_parser.mailjet``
Postmark       ``mailer.webhook.request_parser.postmark``
Sendgrid       ``mailer.webhook.request_parser.sendgrid``
============== ==========================================

.. versionadded:: 6.4

    Support for Brevo, Mailjet, and Sendgrid was added in Symfony 6.4.

Configure the routing:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/framework.yaml
        framework:
            webhook:
                routing:
                    mailer_mailgun:
                        service: 'mailer.webhook.request_parser.mailgun'
                        secret: '%env(MAILER_MAILGUN_SECRET)%'

    .. code-block:: xml

        <!-- config/packages/framework.xml -->
        <framework:config>
            <framework:webhook enabled="true">
                <framework:routing type="mailer_mailgun">
                    <framework:service>mailer.webhook.request_parser.mailgun</framework:service>
                    <framework:secret>%env(MAILER_MAILGUN_SECRET)%</framework:secret>
                </framework:routing>
            </framework:webhook>
        </framework:config>

    .. code-block:: php

        // config/packages/framework.php
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $config): void {
            $config->webhook()
                ->routing('mailer_mailgun')
                ->service('mailer.webhook.request_parser.mailgun')
                ->secret('%env(MAILER_MAILGUN_SECRET)%');
        };

The routing name becomes part of your webhook URL (e.g., ``https://example.com/webhook/mailer_mailgun``). Configure this URL at your mailer provider and store the webhook secret in your environment.

Then create a consumer to handle delivery and engagement events:

.. code-block:: php

    // src/RemoteEvent/MailerWebhookConsumer.php
    namespace App\RemoteEvent;

    use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
    use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
    use Symfony\Component\RemoteEvent\Event\Mailer\MailerDeliveryEvent;
    use Symfony\Component\RemoteEvent\Event\Mailer\MailerEngagementEvent;
    use Symfony\Component\RemoteEvent\RemoteEvent;

    #[AsRemoteEventConsumer('mailer_mailgun')]
    final class MailerWebhookConsumer implements ConsumerInterface
    {
        public function consume(RemoteEvent $event): void
        {
            if ($event instanceof MailerDeliveryEvent) {
                $this->handleDelivery($event);
            } elseif ($event instanceof MailerEngagementEvent) {
                $this->handleEngagement($event);
            }
        }

        private function handleDelivery(MailerDeliveryEvent $event): void
        {
            // Update message status in database, log delivery, etc.
        }

        private function handleEngagement(MailerEngagementEvent $event): void
        {
            // Handle opens, clicks, bounces, etc.
        }
    }

Notifier Webhooks
^^^^^^^^^^^^^^^^^

Receive SMS status notifications from providers:

============ ==========================================
SMS service  Parser service name
============ ==========================================
Twilio       ``notifier.webhook.request_parser.twilio``
Vonage       ``notifier.webhook.request_parser.vonage``
============ ==========================================

Configure similarly to mailers, then consume
:class:`Symfony\\Component\\RemoteEvent\\Event\\Sms\\SmsEvent`:

.. code-block:: php

    // src/RemoteEvent/SmsWebhookConsumer.php
    namespace App\RemoteEvent;

    use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
    use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
    use Symfony\Component\RemoteEvent\Event\Sms\SmsEvent;
    use Symfony\Component\RemoteEvent\RemoteEvent;

    #[AsRemoteEventConsumer('notifier_twilio')]
    final class SmsWebhookConsumer implements ConsumerInterface
    {
        public function consume(RemoteEvent $event): void
        {
            if ($event instanceof SmsEvent) {
                $this->handleSms($event);
            }
        }

        private function handleSms(SmsEvent $event): void
        {
            // Update SMS delivery status in database, etc.
        }
    }


Sending Webhooks
----------------

The Webhook component also enables your application to dispatch webhooks to
external endpoints when your application events occur. This is useful when
building APIs that notify subscribers of important events.

Basic Usage
~~~~~~~~~~~

To send a webhook, dispatch a
:class:`Symfony\\Component\\Webhook\\Messenger\\SendWebhookMessage` via
the Messenger component:

.. code-block:: php

    use Symfony\Component\Webhook\Messenger\SendWebhookMessage;
    use Symfony\Component\Webhook\Subscriber;
    use Symfony\Component\RemoteEvent\RemoteEvent;
    use Symfony\Component\Messenger\MessageBusInterface;

    // In a service or controller

    $subscriber = new Subscriber(
        url: 'https://example.com/webhook/symfony',
        secret: 'your-shared-secret'
    );

    $event = new RemoteEvent(
        name: 'resource.created',
        id: '550e8400-e29b-41d4-a716-446655440000',
        payload: [
            'resource_id' => 12345,
            'email' => 'user@example.com',
            'created_at' => time(),
        ]
    );

    $this->messageBus->dispatch(
        new SendWebhookMessage($subscriber, $event)
    );

The message will be queued and processed by
:class:`Symfony\\Component\\Webhook\\Messenger\\SendWebhookHandler`,
which:

1. Constructs the HTTP request body (JSON-encoded payload)
2. Adds standard headers:

   - ``Webhook-Event``: The event name
   - ``Webhook-Id``: The event ID
   - ``Webhook-Signature``: HMAC-SHA256 signature (configurable) of the
     concatenated event name, ID, and body
   - ``Content-Type``: application/json

3. Signs the request using the subscriber's secret
4. Sends via Symfony's HttpClient component

Webhook Signature
~~~~~~~~~~~~~~~~~

By default, signatures use HMAC-SHA256. The signature header format is:

.. code-block:: text

    Webhook-Signature: sha256=<base64-encoded-hmac>

Receiving endpoints should verify this signature using the shared secret
to ensure webhook authenticity.

Custom Sending Logic
~~~~~~~~~~~~~~~~~~~~

For advanced use cases, you can build custom sending logic using
:class:`Symfony\\Component\\Webhook\\Server\\TransportInterface`.


.. _`MakerBundle`: https://symfony.com/doc/current/bundles/SymfonyMakerBundle/index.html
