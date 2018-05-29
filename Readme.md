# Silex security jwt service provider

[![Build Status](https://travis-ci.org/cnam/security-jwt-service-provider.svg?branch=master)](https://travis-ci.org/cnam/security-jwt-service-provider)
[![Latest Stable Version](https://poser.pugx.org/cnam/security-jwt-service-provider/v/stable)](https://packagist.org/packages/cnam/security-jwt-service-provider) [![Total Downloads](https://poser.pugx.org/cnam/security-jwt-service-provider/downloads)](https://packagist.org/packages/cnam/security-jwt-service-provider) [![Latest Unstable Version](https://poser.pugx.org/cnam/security-jwt-service-provider/v/unstable)](https://packagist.org/packages/cnam/security-jwt-service-provider) [![License](https://poser.pugx.org/cnam/security-jwt-service-provider/license)](https://packagist.org/packages/cnam/security-jwt-service-provider)

This provider is mainly based on [cnam/security-jwt-service-provider](https://github.com/cnam/security-jwt-service-provider), but the dependencies have been updates to Silex 2.2, Pimple 3 and Firebase 5.

## Installation

>  composer require anibalsanchez/security-jwt-service-provider

## Simple example

### Initialise silex application

```php

require_once __DIR__ . '/../../vendor/autoload.php';

$app = new Silex\Application(['debug' => true]);

```

### Create configuration

add config for security jwt

```php

$app['security.jwt'] = [
    'secret_key' => 'Very_secret_key',
    'life_time'  => 86400,
    'options'    => [
        'username_claim' => 'name', // default name, option specifying claim containing username
        'header_name' => 'X-Access-Token', // default null, option for usage normal oauth2 header
        'token_prefix' => 'Bearer',
    ]
];
```

Create users, any user provider implementing interface UserProviderInterface

```php

$app['users'] = function () use ($app) {
    $users = [
        'admin' => array(
            'roles' => array('ROLE_ADMIN'),
            // raw password is foo
            'password' => '5FZ2Z8QIkA7UTZ4BYkoC+GsReLf569mSKDsfods6LYQ8t+a8EW9oaircfMpmaLbPBh4FOBiiFyLfuZmTSUwzZg==',
            'enabled' => true
        ),
    ];

    return new InMemoryUserProvider($users);
};

```

Add config for silex security
 
```php

$app['security.firewalls'] = array(
    'login' => [
        'pattern' => 'login|register|oauth',
        'anonymous' => true,
    ],
    'secured' => array(
        'pattern' => '^.*$',
        'logout' => array('logout_path' => '/logout'),
        'users' => $app['users'],
        'jwt' => array(
            'use_forward' => true,
            'require_previous_session' => false,
            'stateless' => true,
        )
    ),
);

```

Register silex providers

``` php
$app->register(new Silex\Provider\SecurityServiceProvider());
$app->register(new Silex\Provider\SecurityJWTServiceProvider());

```

### Example for authorization and request for protected resources


```php

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Security\Core\Exception\UsernameNotFoundException;
use Symfony\Component\Security\Core\User\InMemoryUserProvider;
use Symfony\Component\Security\Core\User\User;


$app->post('/api/login', function(Request $request) use ($app){
    $vars = json_decode($request->getContent(), true);

    try {
        if (empty($vars['_username']) || empty($vars['_password'])) {
            throw new UsernameNotFoundException(sprintf('Username "%s" does not exist.', $vars['_username']));
        }

        /**
         * @var $user User
         */
        $user = $app['users']->loadUserByUsername($vars['_username']);

        if (! $app['security.encoder.digest']->isPasswordValid($user->getPassword(), $vars['_password'], '')) {
            throw new UsernameNotFoundException(sprintf('Username "%s" does not exist.', $vars['_username']));
        } else {
            $response = [
                'success' => true,
                'token' => $app['security.jwt.encoder']->encode(['name' => $user->getUsername()]),
            ];
        }
    } catch (UsernameNotFoundException $e) {
        $response = [
            'success' => false,
            'error' => 'Invalid credentials',
        ];
    }

    return $app->json($response, ($response['success'] == true ? Response::HTTP_OK : Response::HTTP_BAD_REQUEST));
});

$app->get('/api/protected_resource', function() use ($app){
    return $app->json(['hello' => 'world']);
});

$app->run();

```

Full example in directory tests/mock/app.php

And should for tests correct work silex-security-jwt-provider
 
 
