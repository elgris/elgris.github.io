---
title: "Apitest - test and document that"
author: elgris
date: "2016-02-21"
description: "Public article about ApiTest library for testing APIs"
categories:
    - "tech"
---

Everybody knows that using an API with no documentation is like walking on a minefield with no map.

![danger!](/img/apitest/1.jpg)

In [SeeSaw Labs](http://seesawlabs.com) we build APIs and we're doing our best to provide our customers with such a map :)

But how to write a documentation? Here are 3 general approaches.

## 1. Do it manually
Sounds obvious, right? You write the code, you write documentation for it separately. Sounds easy. Now add several dozens of API endpoints and a dozen of developers into this cocktail. Shake it. Boom - you got Molotov Cocktail! :)

Manually maintained documentation *for code* becomes inaccurate very soon. Sometimes developers just forget updating it. When you keep your documentation control of some VCS you can get merge conflict which is painful for large APIs. It gets even more painful when you have to maintain documentation for systems like Swagger or Postman. C'mon, humans must not maintain documentation for machine, it's called slavery :)

Also it adds extra step to your workflow (which can and should be avoid): "Doc review".

Please note that comments to the code also count as "manual documentation". You have to write them separately, you have to update them and at some point comments begin to smell...

*You got the map, but are you sure it's accurate?*

![you have a map, but is it accurate? !](/img/apitest/2.jpg)

## 2. Generate from code

The most useful approach which adds constraints so you just **cannot** write undocumented code. Documentation generates not from comments, not from annotations, but directly from actual live code. Frameworks like [goa](http://goa.design/) or [go-restful](https://github.com/emicklei/go-restful) help you with this.

Such frameworks are extremely useful when you start to implement your API with them. Unfortunately, if you already have "legacy" code to maintain - usually it's huge pain to refactor it to meet requirements of a framework.

Often they limit your creativity by offering predefined set of tools for, e.g., routing, middleware chains, logging and so on. Frankly speaking, many people don't need fine tuning on transport level since they have a lot of "freedom" (and problems) on business logic level, so predefined components just remove a part of their work and make their life a little bit easier.

Also, such way gives you description, but does not provide examples. You have to forge them by yourself, keep somewhere and maintain well, so people can try your API in action with no frustration. Most probably you will do it manually like described in #1. Wellcome to Hell again :)

Another drawback is that documentation does not ensure that your API will act exactly as documented. You may have absolutely correct example, absolutely descriptive documentation, but after some minor change in business logic your example may become invalid. You just have no tests for your examples

*Now you got some protection, but you're still in danger*

![now you got some protection](/img/apitest/3.jpg)

## 3. Generate from tests
Now we came to 3rd way of getting documentation as well as guaranteed working examples. Automatically. Tests! Holy Grail of software development! :)
This approach works best when you have legacy code. Indeed, no need to refactor the whole codebase at once, you can cover your API with tests, put your test code separately and use it for docs generation. This is the way offered you by [apitest framework](https://github.com/seesawlabs/apitest).

You get other benefits like:
- test coverage which helps a lot with refactoring.
- tests are great source of examples for documentation! At the end you get docs that are "alive", clickable and can be tried by anyone who interested. Examples are generated automatically, thus you get rid of all manual documentation maintenance.

However, this approach is not ideal. First of all, application tests which are used for docs generation (at least with [apitest](https://github.com/seesawlabs/apitest)) are not unit tests. They require running API instance along with all dependencies (like database or external services) or lots of mocks. Such tests can take too much time to run. Also there is not 100% guarantee that your tests produce complete documentation. Some parts may be missing, it depends on how you write code, how you write tests and how you control test coverage metric.

## 4. Combine generation from code and from tests

This is the most reliable approach to get documentation **and** working examples. However, you need a framework that dictates how to write API code and how to write test code in order to keep documentation and examples in sync. This is another story which I'll tell in next article :)

*And here you have full protection!*

![almost a tank](/img/apitest/4.jpg)

# [apitest](https://github.com/seesawlabs/apitest) to the rescue!

Now let's review an example of API test with test framework [apitest](https://github.com/seesawlabs/apitest). Why it's framework? Well, it constrains you. You have to define tests in a specific way. Also, for [apitest](https://github.com/seesawlabs/apitest) there is no "unit-test". API testing is treated as testing of black box with all dependencies enabled/mocked/whatever. That is, you have some input, you have a blackbox and some expected output. [apitest](https://github.com/seesawlabs/apitest) just takes the input, calls a black box and compares output with expected result.

Here's a brief overview how write tests with [apitest](https://github.com/seesawlabs/apitest):

1. For each API endpoint implement interface `IApiTest`.
  1.1. If you need to run some logic before a test or after it, [apitest](https://github.com/seesawlabs/apitest) provides interfaces `ISetuppable` and `ITeardownable` with methods `SetUp()` and `TearDown()`.
2. Populate the test with `[]ApiTestCase` - one test case for every HTTP result code that your API returns.
3. Instantiate or implement `ITestRunner` ([apitest](https://github.com/seesawlabs/apitest) provides basic one) and feed it `[]IApiTest`. Now you have your tests running.
4. Instantiate or implement `IDocGenerator` ([apitest](https://github.com/seesawlabs/apitest) provides generator for Swagger) and feed it `[]IApiTest`. Now you have your documentation generated.

## Define an API
Let's define a small example. A simple API with 2 methods: one of them is extremely simple "Hello world" API and another one - a little bit more complex method queries Github API for details of some user. We can use any toolset we want, let it be `echo` framework

```go
package main

import (
	"errors"
	"net/http"

	"github.com/labstack/echo"
	mw "github.com/labstack/echo/middleware"
	"github.com/octokit/go-octokit/octokit"
)

func main() {
	e := echo.New()

	e.Use(mw.Logger())
	e.Use(mw.Recover())

	// Routes
	e.Get("/hello", hello)
	e.Get("/user/:name", getUser)

	// Start server
	e.Run(":1323")
}

func hello(c *echo.Context) error {
	return c.String(http.StatusOK, "Hello World!\n")
}

func getUser(c *echo.Context) error {
	username := c.Param("name")
	if username == "" {
		return errors.New("parameter 'name' must be provided")
	}

	user, found, err := fetchUserFromGithub(username)
	if err != nil {
		return c.String(http.StatusInternalServerError, err.Error())
	} else if !found {
		return c.String(http.StatusNotFound, "user %s not found", username)
	}

	return c.JSON(http.StatusOK, user)
}

func fetchUserFromGithub(username string) (user *octokit.User, found bool, err error) {
	if username == "BadGuy" {
		return nil, false, errors.New("BadGuy failed me :(")
	}
	client := octokit.NewClient(nil)
	userURL, _ := octokit.UserURL.Expand(octokit.M{"user": username})

	var result *octokit.Result
	user, result = client.Users(userURL).One()

	found = true
	if result.Err != nil {
		err = result.Err
		if responseErr, ok := result.Err.(*octokit.ResponseError); ok {
			found = responseErr.Type != octokit.ErrorNotFound
			if !found {
				err = nil
			}
		}

	}

	return user, found, err
}
```

## Write tests for the API

In order to write tests `apitest`-way you need to implement `IApiTest` interface, 1 implementation for each API endpoint. Each implementation must provide `[]ApiTestcase`, 1 test case for each HTTP response code.

Let's cover our 2 API endpoints with tests:
```
type HelloTest struct {
}

func (t *HelloTest) Method() string      { return "GET" }
func (t *HelloTest) Description() string { return "Test for HelloWorld API handler" }
func (t *HelloTest) Path() string        { return "hello" }
func (t *HelloTest) TestCases() []testilla.ApiTestCase {
	return []testilla.ApiTestCase{
		{
			ExpectedHttpCode: 200,
			ExpectedData:     []byte("Hello World!\n"),
		},
	}
}


type GetUserTest struct {
}

func (t *GetUserTest) Method() string      { return "GET" }
func (t *GetUserTest) Description() string { return "Test for GetUser API handler" }
func (t *GetUserTest) Path() string        { return "user/{username}" }

func (t *GetUserTest) TestCases() []testilla.ApiTestCase {
	elgrisCreatedAt := time.Date(2012, time.June, 29, 11, 57, 38, 0, time.UTC)
	elgrisUpdatedAt := time.Date(2015, time.December, 27, 19, 33, 41, 0, time.UTC)

	return []testilla.ApiTestCase{
		{
			Description: "Successful getting of user details",
			PathParams: testilla.ParamMap{
				"username": testilla.Param{Value: "elgris"},
			},

			ExpectedHttpCode: 200,
			ExpectedData: octokit.User{
				AvatarURL:         "https://avatars.githubusercontent.com/u/1905821?v=3",
				Blog:              "http://elgris-blog.blogspot.com/",
				CreatedAt:         &elgrisCreatedAt,
				UpdatedAt:         &elgrisUpdatedAt,
				EventsURL:         "https://api.github.com/users/elgris/events{/privacy}",
				Followers:         10,
				FollowersURL:      "https://api.github.com/users/elgris/followers",
				Following:         3,
				FollowingURL:      "https://api.github.com/users/elgris/following{/other_user}",
				GistsURL:          "https://api.github.com/users/elgris/gists{/gist_id}",
				Hireable:          true,
				HTMLURL:           "https://github.com/elgris",
				ID:                1905821,
				Location:          "Saint Petersburg, Russia",
				Login:             "elgris",
				Name:              "elgris",
				OrganizationsURL:  "https://api.github.com/users/elgris/orgs",
				PublicRepos:       24,
				ReceivedEventsURL: "https://api.github.com/users/elgris/received_events",
				ReposURL:          "https://api.github.com/users/elgris/repos",
				StarredURL:        "https://api.github.com/users/elgris/starred{/owner}{/repo}",
				SubscriptionsURL:  "https://api.github.com/users/elgris/subscriptions",
				Type:              "User",
				URL:               "https://api.github.com/users/elgris",
			},
		},
		{
			Description: "404 error in case user not found",
			PathParams: testilla.ParamMap{
				"username": testilla.Param{Value: "someveryunknown"},
			},

			ExpectedHttpCode: 404,
			ExpectedData:     []byte("user someveryunknown not found"),
		},
		{
			Description: "500 error in case something bad happens",
			PathParams: testilla.ParamMap{
				"username": testilla.Param{Value: "BadGuy"},
			},

			ExpectedHttpCode: 500,
			ExpectedData:     []byte("BadGuy failed me :("),
		},
	}
}
```

We need a test runner in order to run tests and doc generator to generate swagger specification:

```start_test.go
package main

import (
   "testing"

   "github.com/go-swagger/go-swagger/spec"
   "github.com/seesawlabs/testilla"
)

func TestApi(t *testing.T) {
   tests := []testilla.IApiTest{
      &HelloTest{},
      &GetUserTest{},
   }

   runner := testilla.NewRunner("http://127.0.0.1:1323/")
   runner.Run(tests, t)

   if !t.Failed() {
      seed := spec.Swagger{}
      seed.Host = "127.0.0.1:1323"
      seed.Produces = []string{"application/json"}
      seed.Consumes = []string{"application/json"}
      seed.Schemes = []string{"http"}
      seed.Info = &spec.Info{}
      seed.Info.Description = "Our very little example API with 2 endpoints"
      seed.Info.Title = "Example API"
      seed.Info.Version = "0.1"
      seed.BasePath = "/"

      generator := testilla.NewSwaggerYmlGenerator(seed)

      doc, err := generator.Generate(tests)
      if err != nil {
         t.Fatalf("could not generate docs: %s", err.Error())
      }

      t.Log(string(doc))
   }
}
```

**Note**
The example is provided in [apitest repo](https://github.com/seesawlabs/apitest/tree/master/example]

Now if we start our API on `127.0.0.1:1323` and run `go test`, documentation will be generated and spit in stdout in case of success.

## Extensibility of `apitest`

[apitest](https://github.com/seesawlabs/apitest) gives you an idea, set of interfaces and a bunch of things that can be used as examples. But nothing limits your freedom of imagination! You don't like Swagger? Implement `IDocGenerator` for Postman or API Blueprint or RAML, whatever! Have better idea for test runner? Just implement your very own `ITestRunner`. Just don't forget to send us a pull request, so everybody can enjoy your generator :). Contributions are more than welcome!

## Conclusions

We described ways of building and maintaining documentation for API. Obviousely, doing this job manually is... well... manual, boring job :). Generating docs out of code or tests is a way more efficient and less error-prone. Which way to go? Use API source code only? Or use tests as a source of docs? Or combine both approaches? It's up to you, but you should know that in `SeeSawLabs` we believe that humans must not be slaves for machines, so we give you [apitest](https://github.com/seesawlabs/apitest) and encourage you to automate as much as you can :)