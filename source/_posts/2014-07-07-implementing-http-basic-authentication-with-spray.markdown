---
layout: post
title: "Implementing HTTP Basic authentication with Spray"
date: 2014-07-07 15:15:00 +0100
comments: true
categories: Scala, Akka, Spray
---

This post comes from my answer to.

I've been working with [Spray](http://spray.io) for almost a year now and have been thinking of blogging about some of the things I've found out during that time. When  [this question](http://stackoverflow.com/questions/24501167/spray-authentication-method-with-basicauth/24511242#24511242) came up on StackOverflow I realized I had most of a blog post there, so here's my attempt at explaining how to make HTTP Basic authentication work with Spray.

When I was implementing authentication in my REST API, I first came upon the [Authentication and Authorization](https://github.com/spray/spray/wiki/Authentication-Authorization) page on the Spray wiki. That page says it' applies for Spray 0.9.0, however, I've found most of it still applies as of Spray 1.3.1 running on Akka 2.3.4.

So the first thing to understand is that there are two parts to the process. *Authentication* validates the user's credentials (i.e, login and password, or perhaps an SSL client certificate) and creates some kind of object which carries the user's information, while *authorization* uses the data contained within that object to validate whether the user's action is allowed.

The first thing I did was create the class that will carry forward the authenticated user's info. This class, called, oddly enough, *AuthInfo* looks like this:

    case class AuthInfo(user: ApiUser) {
      def hasPermission(permission: String) = {
      	  // Code to verify whether user has the given permission      }
    }

The `ApiUser` class is the class that contains a user's information. I decided to create a separate case class to separate user information (name, email, etc.) from the details of authorization. I am currently using Strings to identify permissions, although it is not recommended (search for "stringly typed" on your favorite search engine to find out why). This scheme can be as simple as always returning `true` (i.e., all authenticated users have access to all URIs) or can be extended to provide a full-fledged [RBAC](http://en.wikipedia.org/wiki/Role-based_access_control) scheme

Now let's move on to the authentication step. According to [RFC 2617](https://tools.ietf.org/html/rfc2617), when using HTTP Basic authentication the client must send an `Authorization` header such as the following:

    Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==

The string after `Basic` contains the userId and password, separated by a colon and encoded using Base64. If the `Authorization` header is not provided, or if the userid/password combination is incorrect, the server should reply with a `401 Unauthorized` status code, with the `WWW-Authenticate` header containing a *realm*.  The realm is just a String to identify the server. Spray takes care of that automatically when using the `authenticate` directive.

According to [the Spray documentation](http://spray.io/documentation/1.2.1/spray-routing/security-directives/authenticate/), the `authenticate` directive can receive either a `Future[Authentication[T]]` or a `ContextAuthenticator[T]`. A `ContextAuthenticator` is simply a method with the following signature: `RequestContext => Future[Authentication[T]]`, so if you need to authenticate based on request data (in this case, the `Authorization` header) you need to use the second option. Spray already provides a `BasicAuth` object to do HTTP Basic authentication, but you need to provide a function to do the authentication itself (i.e., validating the user and password). I found it most flexible to create a trait that can be mixed into any routes or into the Actor that runs the routes. This is what I came up with:

    trait Authenticator {
      def basicUserAuthenticator(implicit ec: ExecutionContext): AuthMagnet[AuthInfo] = {
        def validateUser(userPass: Option[UserPass]): Option[AuthInfo] = {
          for {
            p <- userPass
            user <- Repository.apiUsers(p.user)
            if user.passwordMatches(p.pass)
          } yield AuthInfo(user)
        }
    
        def authenticator(userPass: Option[UserPass]): Future[Option[AuthInfo]] = Future { validateUser(userPass) }
    
        BasicAuth(authenticator _, realm = "Private API")
      }
    }

In my application, `Repository.apiUsers` is an object with an `apply` method that receives the user's login and returns an `Option[ApiUser]`. I use [scala-bcrypt](https://github.com/t3hnar/scala-bcrypt) to hash the user's password.

    import com.github.t3hnar.bcrypt._
    import org.mindrot.jbcrypt.BCrypt

    case class ApiUser(login: String,
                       hashedPassword: Option[String] = None) {
      def withPassword(password: String) = copy (hashedPassword = Some(password.bcrypt(generateSalt)))
      
      def passwordMatches(password: String): Boolean = hashedPassword.exists(hp => BCrypt.checkpw(password, hp))
    }

With this, I can mix the `Authenticator` trait into the Actor that runs the routes, and use the `authenticate` directive in those routes that need to be authenticated. `authenticate` will pass the `AuthInfo` object generated by the `validateUser` method as a parameter into its closure, and you can then use it wherever you need that info in your routes.

    runRoute(
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
      }
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

I hope this has been useful. At some point I want to extend this to use multiple authentication schemes (starting with SSL Client Certificate and possibly continuing on to use OAuth), as well as extricate the code from the application I am working on and perhaps creating a reusable library, but that will be for the future. For now, enjoy!
