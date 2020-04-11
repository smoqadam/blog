---
title: "How to Authenticate User in Symfony 5 by Jwt"
date: 2020-04-11T00:11:44+08:00
draft: false 
---

## Introduction {#introduction}

Nowadays, when we are talking about web development, regardless of the type of application or the programming language, one of the first things that come to mind is how to authenticate users. There are many types of authentication ways for this purpose such as login form, oAuth, JWT, API token, etc. 

Reliability, security, easy to use and widely supported in many platform and languages make JWT one of the most popular authentication protocols in the web ecosystem.

In this tutorial, we will learn how to implement JWT in Symfony 5 by using the **firebase/php-jwt** package and AbstractGuardAuthenticator class. There are some bundles or packages already out there like **lexik/LexikJWTAuthenticationBundle** that we can use but at the end of this tutorial, we will learn how can we implement and use Authentication Guard which in Symfony.



<!--more-->
## Table of content



- [Prerequisites](#prerequisites)

- [Install and configure Symfony](#install-and-configure-symfony)


  -   [Install necessary packages](#install-necessary-packages)


  -   [User entity](#user-entity)


  -  [Create AuthController](#create-authcontroller)


  -  [Register](#register)


  -  [Login](#login)


  -  [Result](#result)

- [Authenticate User by Guard Authenticator](#authenticate-user-by-guard-authenticator)


  -  [Methods](#methods)


  -  [start](#start)


  -  [support](#support)


  -  [getCredentials](#getcredentials)


  -  [getUser](#getuser)


  -  [onAuthenticationFailure](#onauthenticationfailure)


  -  [onAuthenticationSuccess](#onauthenticationsuccess)


  -  [supportsRememberMe](#supportsrememberme)


  -  [Configuration](#configuration)

- [ApiController](#apicontroller)

- [Summary](#summary)







## Prerequisites {#prerequisites}



*   Basic knowledge about JWT (read about it on [https://jwt.io](https://jwt.io) )
*   The composer must be installed on your machine
*   Be able to use curl command or postman


## Install and configure Symfony {#install-and-configure-symfony}

The Symfony team introduced Symfony installation’s binary which helps us to create a new symfony project skeleton, run a webserver. Go to [this](https://symfony.com/download) link, download and install it. 

Use the following command to create a new Symfony project:


```shell
symfony new jwt-tut
```


### **Install necessary packages** {#install-necessary-packages}

Run the following commands to install necessary packages.


```bash
composer require make
composer require orm
composer require security
composer require firebase/php-jwt
composer require doctrine/annotations
```



### **User entity** {#user-entity}

Whatever authentication system we are going to use, we will always need a User entity even if we don’t want to store user’s data in the database. 

<code>make<strong> </strong></code>command makes it easy to create a new User entity:


```bash
./bin/console make:user
```


after running this command and answering a few questions, the User class will be created under `src/Entity/User.php`.

Notice: If you want to create User entity, make sure to implement UserInterface like below:


```php
class User implements UserInterface
{
	/**
 	* @see UserInterface
 	*/
	public function getSalt()
	{
    		// not needed when using the "bcrypt" algorithm in security.yaml
	}

	/**
 	* @see UserInterface
 	*/
	public function eraseCredentials()
	{
    		// If you store any temporary, sensitive data on the user, clear it here
    		// $this->plainPassword = null;
	}

}
```


Open .env file in your root directory and edit DATABASE_URL then run following commands to create the migration and the table:


```bash
./bin/console make:migration

./bin/console doctrine:migrations:migrate
```



### **Create AuthController** {#create-authcontroller}

So far we have a User class and we installed necessary packages. It’s time to implement register and login functionality.

Create AuthController by running this command:


```bash
./bin/console make:controller AuthController
```


It will generate AuthController.php under src/Controller directory. 


#### **Register** {#register}

Register doesn’t have many things to explain except UserPasswordEncoderInterface which is responsible for encrypting the user’s password. 

To make this interface work, open config/packages/security.yaml and add the encoder section into it. If encoders already exist just change the algorithm to bcrypt. 


```yaml
security:
    # ...
    encoders:
        App\Entity\User:
            algorithm: bcrypt
```


Then add the register method to AuthController as below:

namespace App\Controller;


```php
use App\Entity\User;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Annotation\Route;
use Symfony\Component\Security\Core\Encoder\UserPasswordEncoderInterface;

class AuthController extends AbstractController
{

    /**
     * @Route("/auth/register", name="register", methods={"POST"})
     */
    public function register(Request $request, UserPasswordEncoderInterface $encoder)
    {
        $password = $request->get('password');
        $email = $request->get('email');
        $user = new User();
        $user->setPassword($encoder->encodePassword($user, $password));
        $user->setEmail($email);
        $em = $this->getDoctrine()->getManager();
        $em->persist($user);
        $em->flush();
        return $this->json([
            'user' => $user->getEmail()
        ]);
    }

}
```



#### **Login ** {#login}

To make our JWT token secure, we need to sign it with a secret key and an algorithm. And of course, because we want to use this secret key in several parts of our application, we must put it inside one of the configuration files. Open _config/services.yaml_ and add this line under the parameters section:

    


```yaml
parameters:
    jwt_secret: SOME_SECRET
```


> NOTE: At this point please make sure you are using a strong secret key that no one can guess. 

Add the login method to the AuthController:


```php
/**
 * @Route("/auth/login", name="login", methods={"POST"})
 */
public function login(Request $request, UserRepository $userRepository, UserPasswordEncoderInterface $encoder)
{
        $user = $userRepository->findOneBy([
                'email'=>$request->get('email'),
        ]);
        if (!$user || !$encoder->isPasswordValid($user, $request->get('password'))) {
                return $this->json([
                    'message' => 'email or password is wrong.',
                ]);
        }
       $payload = [
           "user" => $user->getUsername(),
           "exp"  => (new \DateTime())->modify("+5 minutes")->getTimestamp(),
       ];


        $jwt = JWT::encode($payload, $this->getParameter('jwt_secret'), 'HS256');
        return $this->json([
            'message' => 'success!',
            'token' => sprintf('Bearer %s', $jwt),
        ]);
}
```


First we authorized the user by email and password. 

Notice how we used UserPasswordEncoderInterface to validate the user's password. 

Then we created a payload and by using `JWT::encode` from <code>firebase/php-jwt<strong> </strong></code>library we generated a JWT token. This method accepts three parameters:



1. **payload**: is the data you want to send to the client. There are some predefined optional data you can send to the client but here we send the user's email and expirations time which is 5 minutes.
2. **secret**: this is the key to sign the jwt token and it uses to authorize the token
3. **algorithm**: this parameter is optional and the default is set to HS256 but you can use other types of algorithms such as 'ES256', 'HS256', 'HS384', 'HS512', 'RS256', etc.

JWT::encode returns a base64 encoded token which will be used for authentication.


#### Result {#result}

Let's do some tests and see where we are. 

Run the webserver by using `./bin/console server:start` or `symfony server:start`


```bash
$ symfony server:start
```


Then send a request either by Postman or curl to [http://localhost:8000/register](http://localhost:8000/register) 


```bash
$ curl -L -X POST 'http://127.0.0.1:8000/auth/register?email=test@example.com&password=123123'
```


it should return the following json:


```json
{
    "email":"test@example.com"
}
```


Then try to login with the credential you used in the register like this:


```bash
$ curl -L -X POST 'http://127.0.0.1:8000/auth/login' -F 'email=test@example.com' -F 'password=123123'
```


It should return something like this:


```json
{
   "message":"success",
   "token":"Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoidGVzdEBleGFtcGxlLmNvbSJ9.301menCs_ULON3XsQETjUWsUSV2zGZiZztKmWmt18IM"
}
```


Yaay! we did it. If you want to see what is inside your token go to jwt.io and paste the token in Encoded field.


## Authenticate User by Guard Authenticator {#authenticate-user-by-guard-authenticator}

Symfony has an abstract class called AbstractGuardAuthenticator which makes our life easier when it comes to creating authentication for our app. It has several methods that we need to implement to make the authentication work.

Create a class and call it JwtAuthenticator.php under src/Security directory. 


```php
namespace App\Security;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\DependencyInjection\ParameterBag\ContainerBagInterface;
use Symfony\Component\Security\Guard\AbstractGuardAuthenticator;

class JwtAuthenticator
{
    private $em;
    private $params;

    public function __construct(EntityManagerInterface $em, ContainerBagInterface $params)
    {
        $this->em = $em;
        $this->params = $params;
    }
}
```


EntityManagerInterface will be used to connect to the database to get the user data. The ContainerBagInterface is using to read configuration files to read _jwt_secret_ in _services.yaml_ which we created before.

Now extend your class from AbstractGuardAuthenticator and implement the following methods:


```php
namespace App\Security;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\DependencyInjection\ParameterBag\ContainerBagInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Exception\AuthenticationException;
use Symfony\Component\Security\Core\User\UserInterface;
use Symfony\Component\Security\Core\User\UserProviderInterface;
use Symfony\Component\Security\Guard\AbstractGuardAuthenticator;

class JwtAuthenticator extends AbstractGuardAuthenticator
{
    private $em;
    private $params;

    public function __construct(EntityManagerInterface $em, ContainerBagInterface $params)
    {
        $this->em = $em;
        $this->params = $params;
    }

    public function start(Request $request, AuthenticationException $authException = null)
    {
        
    }

    public function supports(Request $request)
    {
        
    }

    public function getCredentials(Request $request)
    {
        
    }

    public function getUser($credentials, UserProviderInterface $userProvider)
    {
        
    }

    public function checkCredentials($credentials, UserInterface $user)
    {
        
    }

    public function onAuthenticationFailure(Request $request, AuthenticationException $exception)
    {
        
    }

    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $providerKey)
    {
        
    }

    public function supportsRememberMe()
    {
        
    }
}
```


Don’t panic, we will go through all these methods to see how they can help us to get our job done.


### Methods {#methods}


#### start {#start}

Whenever a user wants to access a URL or resources that need authentication, but the authentication details were not sent, this method will run. In return, we must return a Response object with a 401 status code.


```php
public function start(Request $request, AuthenticationException $authException = null) \
{ 
    $data = [ 
        'message' => 'Authentication Required'
    ];
    return new JsonResponse($data, Response::HTTP_UNAUTHORIZED);
}
```


#### support {#support}

This method checks whether the current request supports the authentication or not. As you may know, JWT is using the Authorization header to transfer the token between server and client. 

If return false, authentication will be skipped. 


```php
public function supports(Request $request)
{
    return $request->headers->has('Authorization');
}
```



#### getCredentials {#getcredentials}

This method returns the value of the Authorization header which is the token we return when user login.


```php
public function getCredentials(Request $request)
{
        return $request->headers->get('Authorization');
}
```


This method returns the value of the Authorization header which is the token we return when user login.


#### getUser {#getuser}

This method is responsible to validate JWT Token and authenticate the user by the credentials' value which is returned from the getCredential method and it must return a User entity object or AuthenticationException. In this method, 



1. We removed _Bearer_ from our token
2. Then decoded the JWT token by _JWT::decode_ method. _JWT::decode _receives three arguments_, _first argument is the token, then the secret key and then the algorithm we used to encode the token. If decoding finished successfully it will return the payload otherwise it will throw one of these exceptions:

```phpDoc
* @throws UnexpectedValueException Provided JWT was invalid
* @throws SignatureInvalidException Provided JWT was invalid because the signature verification failed
* @throws BeforeValidException Provided JWT is trying to be used before it's eligible as defined by 'nbf'
* @throws BeforeValidException Provided JWT is trying to be used before it's been created as defined by 'iat'
* @throws ExpiredException Provided JWT has since expired, as defined by the 'exp' claim
```



That’s why we put our code inside try/catch to catch exceptions. 


```php
public function getUser($credentials, UserProviderInterface $userProvider)
{
        try {
            $credentials = str_replace('Bearer ', '', $credentials);
            $jwt = (array) JWT::decode(
                              $credentials, 
                              $this->params->get('jwt_secret'),
                              ['HS256']
                            );
            return $this->em->getRepository(User::class)
                    ->findOneBy([
                            'email' => $jwt['user'],
                    ]);
        }catch (\Exception $exception) {
                throw new AuthenticationException($exception->getMessage());
        }
}
```



#### onAuthenticationFailure {#onauthenticationfailure}

As I mentioned before, this method will be called if we throw an AuthenticationException from the getUser method. This must return a Response object.


```php
public function onAuthenticationFailure(Request $request, AuthenticationException $exception)
{
        return new JsonResponse([
                'message' => $exception->getMessage()
        ], Response::HTTP_UNAUTHORIZED);
}
```



#### onAuthenticationSuccess {#onauthenticationsuccess}

This method will call if the authentication were successful. however, in our example we don’t need to return anything.


```php
public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $providerKey)
{
    return;
}
```



#### supportsRememberMe {#supportsrememberme}

Since this is a stateless API we don’t need “remember me” functionality. 


```php
public function supportsRememberMe()
{
    return false;
}
```



### Configuration {#configuration}

We are almost there. So far, we have implemented register, login and the JWT authentication guard. But to make the authentication work, we need to tell Symfony to use our authentication class. 

For this purpose, open the `config/packages/security.yaml` and change main section under firewall as below:


```yaml
security:
    # ...
    firewalls:
            main:
                    pattern: ^/api
                    guard:
                        authenticators:
                                - App\Security\JwtAuthenticator
```


We added a pattern key with _^/api_ value to protect all routes which are starting with _/api_ such as /api/user, /api/posts, etc.

Under the guard section, we have specified our authenticator class. We can have more than one authenticator class but in most cases, one authenticator will do the job. 


## ApiController {#apicontroller}

Our job is almost finished, now it’s time to see if it works or not! 

Create another controller and call it ApiController


```bash
$ ./bin/console make:controller ApiController
```


open _src/Controller/ApiController.php_ and add a new method:


```php
/**
* @Route("/api/test", name="testapi")
*/
public function test()
{
      return $this->json([
              'message' => 'test!',
       ]);
}
```


As you can see, the route starts with /api so we should expect any request to this route with a **valid** token should return `{"message": "test!"}` with status code 200, otherwise, it should return an error message with status code 401.

Run the following command to see the result:


```yaml
$ curl -L -X GET 'http://127.0.0.1:8000/api/test' -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoidGVzdEBleGFtcGxlLmNvbSJ9.301menCs_ULON3XsQETjUWsUSV2zGZiZztKmWmt18IM'
 
{"message": "test!"}

```


Awesome! we got what we expected. 


## Summary {#summary}

We reached the end of this article. We learned how to install Symfony, install necessary packages and how to use AbstractGuardAuthenticator to authenticate users. Actually, we can use AbstractGuardAuthenticator in any kind of authentication. To use this code in production, we need to do some other things such as check the token’s expiration, change the error messages, or make a separate bundle to be able to use it in future. The alternative solution is to use one of the opensource bundles such as **lexik/LexikJWTAuthenticationBundle.**

**Anyway, Thanks for reading this article, if you have any thoughts or feedback, I would be super happy to hear it. Please mention it in the github [issues](https://github.com/smoqadam/blog/issues) or message me on [twitter](https://twitter.com/saeedMoqadam).**



