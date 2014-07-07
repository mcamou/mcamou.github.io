---
layout: post
title: "Implementing HTTP Basic authentication with Spray"
date: 2014-07-07 15:15:00 +0100
comments: true
categories: [Scala, Akka, Spray]
---

I've been working with [Spray](http://spray.io) for almost a year now and have been thinking of blogging about some of the things I've found out during that time. After answering [this question](http://stackoverflow.com/questions/24501167/spray-authentication-method-with-basicauth/24511242#24511242) on StackOverflow I realized I had most of a blog post there, so here's my attempt at explaining how to make HTTP Basic authentication work with Spray.

When implementing authentication for a REST API, I first came upon the [Authentication and Authorization](https://github.com/spray/spray/wiki/Authentication-Authorization) page on the Spray wiki. That page says the documentation is for for Spray 0.9.0, however, I've found most of it still applies as of Spray 1.3.1 running on Akka 2.3.4.

The first thing to understand is that there are two parts to the process:

* *Authentication*, which validates the user's credentials (i.e, login and password, or perhaps an SSL client certificate) and creates some kind of object which carries the user's information.

* *Authorization*, which uses the data contained within that object to validate whether the user's action is allowed.

First of all, I have an `ApiUser` class that holds the user's login and password. It could also hold additional info such as the user's name, email, etc. I use [scala-bcrypt](https://github.com/t3hnar/scala-bcrypt) to hash the user's password, and added a `withPassword` method to set the user's password (which is never stored unencrypted) and a `passwordMatches` method which verifies if the user's password matches the given password. `hashedPassword` is an `Option[String]` so I can set it to `None` to disable a user's login.

    import com.github.t3hnar.bcrypt._
    import org.mindrot.jbcrypt.BCrypt

    case class ApiUser(login: String,
                       hashedPassword: Option[String] = None) {
      def withPassword(password: String) = copy (hashedPassword = Some(password.bcrypt(generateSalt)))
      
      def passwordMatches(password: String): Boolean = hashedPassword.exists(hp => BCrypt.checkpw(password, hp))
    }


Once I had a class to hold the user's information, I created another class to carry the authenticated user's information and verify whether the user has permission to perform an action. I decided to create a separate case class to separate user information (name, email, etc.) from the details of authorization.

    class AuthInfo(val user: ApiUser) {
      def hasPermission(permission: String) = {
      	  // Code to verify whether user has the given permission      }
    }

I am currently using Strings to identify permissions, although it is not recommended (search for "stringly typed" on your favorite search engine to find out why). This scheme can be as simple as always returning `true` (i.e., all authenticated users have access to all URIs) or can be extended to provide a full-fledged [RBAC](http://en.wikipedia.org/wiki/Role-based_access_control) scheme

With this set up, we can go ahead with the authentication step. According to [RFC 2617](https://tools.ietf.org/html/rfc2617), when using HTTP Basic authentication the client must send an `Authorization` header such as the following:

    Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==

The string after `Basic` contains the userId and password, separated by a colon and encoded using Base64. If the `Authorization` header is not provided, or if the userid/password combination is incorrect, the server should reply with a `401 Unauthorized` status code, with the `WWW-Authenticate` header containing a *realm*.  The realm is just a String to identify the server. Spray provides a `BasicAuth` class to take care of all that automatically in conjunction with the `authenticate` directive.

According to [the Spray documentation](http://spray.io/documentation/1.2.1/spray-routing/security-directives/authenticate/), the `authenticate` directive can receive either a `Future[Authentication[T]]` or a `ContextAuthenticator[T]`. A `ContextAuthenticator` is simply a method with that receives a `RequestContext` (which contains all the request information) and returns a `Future[Authentication[T]]`. Since we need to authenticate based on request data (in this case, the `Authorization` header), we need to use the second option.

Spray provides a `BasicAuth` class that extends `Authentication[T]` and takes care of decoding the login and password from the `Authorization` header, but we need to provide a function to do the authentication itself (i.e., validating the user and password) and return a `Future[Option[T]]`. In our case, `T` is our `AuthInfo` class, so it should return a `Future` containing `Some(authInfo)` if the login and password are valid, or `None` if they aren't. I found it most flexible to create a trait that can be mixed into any routes or into the Actor that runs the routes.

    trait Authenticator {
      def basicUserAuthenticator(implicit ec: ExecutionContext): AuthMagnet[AuthInfo] = {
        def validateUser(userPass: Option[UserPass]): Option[AuthInfo] = {
          for {
            p <- userPass
            user <- Repository.apiUsers(p.user)
            if user.passwordMatches(p.pass)
          } yield new AuthInfo(user)
        }
    
        def authenticator(userPass: Option[UserPass]): Future[Option[AuthInfo]] = future { validateUser(userPass) }
    
        BasicAuth(authenticator _, realm = "Private API")
      }
    }

In my application, `Repository.apiUsers` is an object with an `apply` method that receives the user's login and returns an `Option[ApiUser]`. You can replace it with, for example, a Slick query or a call to an LDAP server, depending on your application's needs. One of the nice things about Spray is that, since everything is a `Future`, those requests will happen asynchronously.

With this, we can mix the `Authenticator` trait into the Actor that runs the routes, and use the `authenticate` directive in those routes that need to be authenticated. `authenticate` will pass the `AuthInfo` object generated by the `validateUser` method as a parameter into its closure, and you can then use it wherever you need that info in your routes.


    class ApiActor extends Actor
                           with ActorLogging
                           with Authenticator {
      implicit val executionContext = context.dispatcher
    
      override def receive = {
        runRoute(
          pathEndOrSingleSlash {
            // Any user, authenticated or not, can enter here
            complete("System OK")
          } ~
          pathPrefix("api") {
            authenticate(basicUserAuthenticator) { authInfo =>
              path("private") {
                get {
                  // All authenticated users can enter here
                  complete(s"Hi, ${authInfo.user.login}")
                }
              }
            }
          }
        )
      }

What if you need more fine-grained authorization? For example, you might want to allow all users to GET a particular URI, but only some users to PUT or POST. That is what the `authorize` directive is for. `authorize` receives a by-name that returns `true` if the user is allowed to perform the action, and `false` if she isn't. You can use the `authInfo` object to check that.

        authenticate(basicUserAuthenticator) { authInfo =>
          path("private") {
            get {
                // All authenticated users can enter here
                complete(s"Hi, ${authInfo.user.login}")
              }
              post {
                authorize(authInfo.hasPermission("post") {
                  // Only those users that have the "post" permission will be allowed in here
                }              }
            }
          }

If the authorization is unsuccessful, Spray will return the HTTP `403 Forbidden` status code.

I hope this has been useful. At some point I would like to extend this to use multiple authentication schemes (starting with SSL Client Certificate and possibly continuing on to use OAuth), as well as extricate the code from the application I am working on and perhaps create a reusable library, but that will be for the future. For now, enjoy!
