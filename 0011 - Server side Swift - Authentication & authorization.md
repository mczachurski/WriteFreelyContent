# Server side Swift - Authentication & authorization

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0026.png)

> This is the article created at Mar 2, 2018 and moved from Medium.

My previous articles described how I realised MVC pattern in server side Swift HTTP application, how I added support for configuration settings and ORM framework and implement repository pattern. Now we will add authentication and authorization to our API.
<!--more-->

---

First we have to talk about two mechanisms that very often are mistaken: **_authentication_** and **_authorization_**.

**Authentication** is the process of ascertaining that somebody really is who he claims to be. In most cases user have to enter his login and password (or use TouchID, FaceID or other biometric mechanism).

**Authorization** refers to rules that determine who is allowed to do what. So system have to store our permissions _(and check what we are allowed to do)._ It have to verify if we are authorized to access resource or perform specific operation.

Here I will describe how we can introduce authentication and authorization (to endpoints) in our server side Swift application which based on Perfect Server.

---

#### Authentication

Authentication in API is a little bit different then in web applications. In web application we can store information about user in _session_ and use them during all requests to the server. In API world we have other mechanism that is now very popular: JSON Web Tokens (JWT).

In that mechanism our API will create a token when user enter correct email and password. And in further communication user (client) can use that token to authenticate himself. Whole algorithm is described in [RFC 7519](https://tools.ietf.org/html/rfc7519). There is also great page: [https://jwt.jo](https://jwt.io), which allows you to decode, verify and generate JWT tokens. Additionally on that page you can find list of implementation of JWT algorithm. Of course there are also implementations written in Objective-C and Swift.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0027.png)

Following two implementations are mentioned on above sites.

- [kylef/JSONWebToken.swift](https://github.com/kylef/JSONWebToken.swift)
- [vapor/jwt](https://github.com/vapor/jwt)

However I will use implementation which wasn’t mentioned on that site and it was created by the same team which created Perfect Server: [PerfectlySoft/Perfect-Crypto](https://github.com/PerfectlySoft/Perfect-Crypto).

You can find on above page example how to use their JWT implementation. Creating a new token looks like on following snippet:

```swift
let name = "John Doe"
let payload = ["name": name, "admin": true] as [String : Any]
let secret = "secret"

guard let jwt = JWTCreator(payload: payload) else {
    return
}

let token = try jwt.sign(alg: .hs256, key: secret)
```

Verifing token (and retrieving information from token) is also really simple:

```swift
guard let jwt = JWTVerifier(token) else {
    return
}

try jwt.verify(algo: .hs256, key: HMACKey(secret))
let fndName = jwt.payload["name"] as? String
```

So let’s look how you can use that framework in your application.

#### Registering account

First we have to prepare endpoint where user will have possiblilities to create his account. My implementation is really simple. I’m reading JSON from request body which looks like this:

```json
{
    "name": "Marcin Czachurski",
    "email": "me@address.com",
    "password": "p@ssw0rd",
    "isLocked": false
}
```

My controller endpoint looks like on below snippet.

```swift
public func register(request: HTTPRequest, response: HTTPResponse) {
    do {
        let registerUserDto = try request.getObjectFromRequest(RegisterUserDto.self)
        let user = registerUserDto.toUser()

        try self.usersService.add(entity: user)

        guard let registeredUser = try self.usersService.get(byId: user.id) else {
            return response.sendNotFoundError()
        }

        let registeredUserDto = UserDto(user: registeredUser)
        return response.sendJson(registeredUserDto)
    }
    catch let error where error is DecodingError || error is RequestError {
        response.sendBadRequestError()
    }
    catch let error as ValidationsError {
        response.sendValidationsError(error: error)
    }
    catch {
        response.sendInternalServerError(error: error)
    }
}
```

First I’m converting JSON to the DTO (`RegisterUserDto`) object in the application. Then we have to convert this object to the object which we store in our database: `let user = registerUserDto.toUser()`. Then by using our `userService` I’m adding new user to the database.

My method which is responsible for adding new user looks like that:

```swift
public func add(entity: User) throws {

    entity.salt = String(randomWithLength: 14)
    entity.password = try entity.password.generateHash(salt: entity.salt)

    if let errors = self.userValidator.getValidationErrors(entity) {
        throw ValidationsError(errors: errors)
    }

    try self.usersRepository.add(entity: entity)
    try self.userRolesRepository.set(roles: entity.roles, forUserId: entity.id)
}
```

During adding new user we have to transform user password. We will create hash from password. It will be impossible to reconstruct real password from hash. First I’m creating `salt` variable. This is variable which stores random alphanumeric characters. This salt will be added to the password and stored also in the database. Thanks to this, we will never have the same password hashes in the database, even if users has the same passwords.

My function for generating password looks like that:

```swift
public func generateHash(salt: String) throws -> String {
    let stringWithSalt = salt + self

    guard let stringArray = stringWithSalt.digest(.sha256)?.encode(.base64) else {
        throw GeneratePasswordError()
    }

    guard let stringHash = String(data: Data(bytes: stringArray, count: stringArray.count), encoding: .utf8) else {
        throw GeneratePasswordError()
    }

    return stringHash
}
```

#### Signing In

Now user can try to sign in to the system. For that suppose I have separate endpoint, which expects following JSON:

```json
{
    "email": "me@address.com",
    "password": "p@ssw0rd"
}
```

Source code of that endpoint looks like on below snippet:

```swift
public func signIn(request: HTTPRequest, response: HTTPResponse) {
    do {
        let signIn = try request.getObjectFromRequest(SignInDto.self)

        guard let user = try self.usersService.get(byEmail: signIn.email) else {
            return response.sendNotFoundError()
        }

        let password = try signIn.password.generateHash(salt: user.salt)
        if password != user.password {
            return response.sendNotFoundError()
        }

        let tokenDto = try self.prepareToken(user: user)
        return response.sendJson(tokenDto)
    }
    catch let error where error is DecodingError || error is RequestError {
        response.sendBadRequestError()
    }
    catch let error as ValidationsError {
        response.sendValidationsError(error: error)
    }
    catch {
        response.sendInternalServerError(error: error)
    }
}
```

First I’m retrieving user from the database by his email. Then I’m generating once again hash of his password (based on `salt` from the database and `password` from JSON). If generated hash is the same as a hash from the database then we have to prepare JWT token for user. For that purpose I created separate method:

```swift
private func prepareToken(user: User) throws -> JwtTokenResponseDto {

    let payload = [
        ClaimsNames.name.rawValue           : user.email,
        ClaimsNames.roles.rawValue          : user.getRolesNames(),
        ClaimsNames.issuer.rawValue         : self.configuration.issuer,
        ClaimsNames.issuedAt.rawValue       : Date().timeIntervalSince1970,
        ClaimsNames.expiration.rawValue     : Date().addingTimeInterval(36000).timeIntervalSince1970
    ] as [String : Any]

    guard let jwt = JWTCreator(payload: payload) else {
        throw PrepareTokenError()
    }

    let token = try jwt.sign(alg: .hs256, key: self.configuration.secret)

    let tokenDto = JwtTokenResponseDto(token: token)
    return tokenDto
}
```

As you can see I’m creating token with some additonal information. We can verify generated tokens on [https://jwt.io](https://jwt.io) site. For example when I’m encoding one of my token I have following results.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0028.png)

I’m storing in token three important things:

- `name` — user identifier, in my case user email
- `roles` — list of roles which user is assigned to
- `exp` — expiration date, after that date token is invalid

All that information will be important during user authorization process.

#### Authorization filter

In all further requests, client application have to include token in requests’s header:

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJ0YXNrZXItc2VydmVyIiwibmFtZSI6Im1hcmNpbi5jemFjaHVyc2tpQGljbG91ZC5jb20iLCJyb2xlcyI6WyJVc2VyIiwiQWRtaW5pc3RyYXRvciJdLCJleHAiOjE1MTk5MzMxNDEuNTg4ODQsImlhdCI6MTUxOTg5NzE0MS41ODg4NH0.6rk_klpGCIEqpWs1zVFAQM6k8bbHP6yHibpmmFxdrZk
```

However we cannot protect all endpoints (actions in the controller). Still we have to have some actions that should be _visible_ for anounymous users, like `signIn` action. For that purposes I extended my method when I’m registering routes. I added aditional parameter: `authorization`, which can have one of three values:

- `anonymous` — all users can use endpoint
- `signedIn` — signed in users can use endpoint
- `inRole([String])` — signed in users with specific roles can use endpoint

Example of registration routes:

```swift
override func initRoutes() {
    self.add(method: .post, uri: "/account/register", authorization: .anonymous, handler: register)
    self.add(method: .post, uri: "/account/signIn", authorization: .anonymous, handler: signIn)
    self.add(method: .post, uri: "/account/changePassword", authorization: .signedIn, handler: changePassword)
}
```

Now we can add special filter which will validate JWT tokens. That filter (`AuthorizationFilter` which inherites from `HTTPRequestFilter`) is executing during all requests to the server (before our controller’s action). Below is code which I have in my application.

```swift
import Foundation
import PerfectHTTP
import PerfectCrypto

public class AuthorizationFilter: HTTPRequestFilter {
    
    private let secret: String
    private let routesWithAuthorization: Routes
    
    public init(secret: String, routesWithAuthorization: Routes) {
        self.secret = secret
        self.routesWithAuthorization = routesWithAuthorization
    }
    
    public func filter(request: HTTPRequest, response: HTTPResponse, callback: (HTTPRequestFilterResult) -> ()) {
        
        guard let _ = self.routesWithAuthorization.navigator.findHandler(uri: request.uri, webRequest: request) else {
            return callback(.continue(request, response))
        }
        
        guard var header = request.header(.authorization) else {
            response.sendUnauthorizedError()
            return callback(.halt(request, response))
        }
        
        guard header.starts(with: "Bearer ") else {
            response.sendUnauthorizedError()
            return callback(.halt(request, response))
        }
        
        do {
            header.removeFirst(7)
            
            guard let jwt = JWTVerifier(header) else {
                response.sendUnauthorizedError()
                return callback(.halt(request, response))
            }
            
            try jwt.verify(algo: .hs256, key: HMACKey(secret))
            try jwt.verifyExpirationDate()

            self.addUserCredentialsToRequest(request: request, jwt: jwt)
            
        } catch is JWTTokenExpirationDateError {
            print("Token expiration date error.")
            response.sendUnauthorizedError()
            return callback(.halt(request, response))
        } catch {
            print("Failed to decode JWT: \(error)")
            response.sendUnauthorizedError()
            return callback(.halt(request, response))
        }
        
        callback(.continue(request, response))
    }

    private func addUserCredentialsToRequest(request: HTTPRequest, jwt: JWTVerifier) {
        if let name = jwt.payload[ClaimsNames.name.rawValue] as? String {
            let userCredentials = UserCredentials(
                name: name, 
                roles: jwt.payload[ClaimsNames.roles.rawValue] as? [String]
            )
            request.add(userCredentials: userCredentials)
        }
    }
}
```

First step is to verify if endpoint which will be executed requires that user has to be signed in to the system. If requires then we have to read from header JWT token and verify it. We have to verify token correctness and expiration date. This two lines of code are responsible for that.

```swift
try jwt.verify(algo: .hs256, key: HMACKey(secret))
try jwt.verifyExpirationDate()
```

If token is valid we can add user information to the request. Thanks to this in our endpoints we will have information about authorized user. For that puposes I’m executing following line.

```swift
self.addUserCredentialsToRequest(request: request, jwt: jwt)
```

Above method adds to the `HTTPRequest.scratchPad` new `UserCredentials` object with user information. We can use that information in our actions in controller.

Of course we have to register our new filter in `HTTPServer`. I’m doing this like that:

```swift
// Authorization.
var routesWithAuthorization = Routes()
routesWithAuthorization.configure(routesWithAuthorization: controllers)

let requestFilters: [(HTTPRequestFilter, HTTPFilterPriority)] = [
    (AuthenticationFilter(secret: configuration.secret, routesWithAuthorization: routesWithAuthorization), HTTPFilterPriority.high)
]

do {
    // Launch the HTTP server.
    try HTTPServer.launch(
        .server(
            name: configuration.serverName,
            port: configuration.serverPort,
            routes: allRoutes,
            requestFilters: requestFilters
        )
    )
} catch {
    fatalError("\(error)") // fatal error launching one of the servers
}
```

It’s important to inject to the filter collection of routes which should be protected.

---

Now we have implemented authentication and authorization to the endpoints in our Swift API application. In my next article I will describe how I solved resource based authorization (authorization to the specific resource/object).

---

All source codes you can find in my GitHub project (branch `authentication`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).