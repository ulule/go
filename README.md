# Go Style Guide

*A mostly reasonable approach to Go*

## Wisdom Quotes

KISS: Keep it simple, stupid

DRY: Don't repeat yourself

Stay [idiomatic](https://golang.org/doc/effective_go.html)

Embrace [12factor](https://12factor.net/) as much as possible

## Package names

Avoid too much level for your packages (3 levels is mostly enough).

Prefer package names based on feature instead of which software they are interacting to.

For example:

* ``redis`` will become ``caching``
* ``rabbitmq`` will become ``brokers`` or ``messaging``

For these kind of packages, you will include a ``dummy`` implementation to facilitate unittests.

An example of ``rabbitmq`` dummy implementation can be an in-memory storage to store events.

For unittests, you will need to rename your package if unittests are in the same level of your logic.

**Example:** ``models`` package will have ``models_test`` a package name at the same level.

```
models/
	users.go
	users_test.go
```

> `models/users.go`

```go
package models
```

> `models/users_test.go`

```go
package models_test
```

## Required development tools

* [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports)
* [golint](https://github.com/golang/lint)
* [gometalinter](https://github.com/alecthomas/gometalinter) which includes previous tools and run them asynchronously
* [gofmt](https://golang.org/cmd/gofmt/) with the ``-s`` option to simplify code

Integrate those tools with your own editor.

**gometalinter** is recommended _-as your default linter-_ with the following configuration:

> `gometalinter.json`

```json
{
  "DisableAll": true,
  "Enable": [
    "lll",
    "unused",
    "misspell",
    "gofmt",
    "dupl",
    "gosimple",
    "ineffassign",
    "errcheck",
    "gas",
    "vet",
    "unconvert",
    "interfacer",
    "deadcode",
    "gocyclo",
    "vetshadow",
    "golint",
    "goconst",
    "staticcheck",
    "varcheck",
    "structcheck"
  ],
  "EnableGC": true,
  "Deadline": "1200s",
  "Concurrency": 1,
  "Vendor": true,
  "VendoredLinters": true,
  "Aggregate": true,
  "Test": true,
  "LineLength": 120,
  "Cyclo": 10,
  "DuplThreshold": 80
}
```

A straightforward example would be:

`gometalinter --config=gometalinter.json ./...`

## Be readable

Give a short but explicit name to your variables, functions and use ``lowerCamelCase`` representation.

```go
// bad: too short
a := "foo@ulule.com"

// bad: too long
userEmailWithRootPermissions := "bar@ulule.com"

// good
userEmail := "foo@ulule.com"
email := "bar@ulule.com"
```

Group your logic within the same block

Short and pure functions, avoid a function which will contain more than 100 lines.

When your code is too complex:

 1. Try to separate things
 2. Try again
 3. Add comments

Name your functions based on their behavior, for example:

A function which will panic and not return an error will be prefixed by ``Must``: [regexp.MustCompile](https://golang.org/pkg/regexp/#MustCompile)

Group packages import, new line between each

 1. Standard library
 2. External packages
 3. Internal packages

In common cases, if you need to rename packages when you are importing them, you are doing it wrong.

```go
import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/mholt/binding"
	"github.com/ulule/deepcopier"
	redis "gopkg.in/redis.v5"
	redisLocker "github.com/ulule/kyu/locker/redis"

	"github.com/ulule/foo/bar"
)
```

**DO NOT** rely on ``.`` import.

## Be secure

Always check users input by using validators ([govalidator](https://github.com/asaskevich/govalidator)) and sanitize them.

Don't [sanitize](https://github.com/microcosm-cc/bluemonday) an input which is already validated, it's useless and you can introduce unexpected behaviors.

> `validators.go`

```go
var PaymentMethods = map[string]string{
	PaymentMethodCreditCard:  "Credit card",
	PaymentMethodCheck:       "Check",
	PaymentMethodPaypal:      "Paypal",
	PaymentMethodDirectDebit: "Direct debit",
	PaymentMethodMaestro:     "Maestro",
}

const (
	ValueError                = "ValueError"
	PaymentMethodErrorMessage = "Payment method is invalid"
)

func PaymentMethod(paymentMethod string) binding.Errors {
	errs := binding.Errors{}

	if _, ok := PaymentMethods[paymentMethod]; !ok {
		errs = append(errs, binding.Error{
			FieldNames:     []string{"payment_method"},
			Classification: ValueError,
			Message:        PaymentMethodErrorMessage,
		})
	}

	return errs
}
```

There is no trusted sources.

Do not assume, verify your output.

Always ``rand.Seed(time.Now().Unix())`` before using [rand](https://golang.org/pkg/math/rand/).

## Be realistic

Your code will fail, use [recover](https://blog.golang.org/defer-panic-and-recover), sentry, alerting, etc.

Internal / external services will also fail, rely on services degradation as much as possible.

* ``caching``: disable cache if your redis server is down
* ``messaging``: rely on a in-memory broker when rabbitmq is down

## Code organization

Be explicit when naming thing.

```go
// bad
package user

func Get()
```

```go
// good
package models

func GetUser()
```

Avoid doing useless tests when your method return a ``bool`` or the same output of a previously called method.

In general, ``else`` clause can be avoided by returning first.

```go
// bad
func UserExists(ctx context.Context, id int) (bool, error) {
	exists, err := store.UserExists(ctx, id)

	if err == nil {
		if exists {
			return true, nil
		} else {
			return false, nil
		}
	}

	return false, err
}

// good
func UserExists(ctx context.Context, id int) (bool, error) {
	exists, err := store.UserExists(ctx, id)
	if err != nil {
		return false, err
	}

	return exists, nil
}

// better
func UserExists(ctx context.Context, id int) (bool, error) {
	return store.UserExists(ctx, id)
}
```

```go
// not so bad
users := map[int]*User{}
exists := false

// not so bad either
var (
	users  = map[int]*User{}
	exists = false
)
```

Don't rely on [zero values](https://tour.golang.org/basics/12), be explicit as much as possible, you can apply
[The Zen of Python](https://www.python.org/dev/peps/pep-0020/).

```go
// very bad
var exists bool // false
var counter int // 0

// good
var (
    exists  = false
    counter = 0
)

// also good
exists  := false
counter := 0
```

Rely on [context](https://golang.org/pkg/context/) everywhere.

We should use [named result parameters](https://golang.org/doc/effective_go.html#named-results) when the type of at
least two successive returned variables is the same.

However, we must avoid [naked return statement](https://tour.golang.org/basics/7).

**bad**

```go
package main

import "fmt"

func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}

func main() {
	fmt.Println(split(17))
}
```

**good**

```go
package main

import "fmt"

func split(sum int) (x int, y int) {
	x = sum * 4 / 9
	return x, sum - x
}

func main() {
	fmt.Println(split(17))
}
```

## Types

Types should be explicit as much as possible:

  * Avoid `interface{}` type.
  * Don't be shy and/or skinflint about interface.
  * Visitor pattern is your best-friend if you can't use an interface, in order to avoid `interface{}` type.

Use generic litterals _(`int`, `float`, etc...)_ **unless you are defining a model**, which should follow these
conventions:

  * Use `int64` type for primary, foreign key and counter.
  * Use `Enumerated` type for enumeration.

```go
// Foobar is ...
type Foobar struct {
    ID        int64 `db:"id"`
    UserID    int64 `db:"user_id"`
    AccountID int64 `db:"account_id"`
    HitCount  int64 `db:"hit_count"`
    MissCount int64 `db:"miss_count"`
    Status    Enumerated `db:"status"`
    Type      Enumerated `db:"type"`
}
```

Also, don't hesitate to use type definitions to create a subtype in order to specify and restrict what you expect:

```go
type Email string
```

## Logging

Only use one of these levels:

* **Debug:** A verbose entry usually used for future debugging.
* **Info:** A general entry about what's going on inside the application.
* **Warning:** A non-critical entry that deserve review.
* **Error:** A critical and high-priority entry that deserve reaction.

Use structured logging:

```go
// bad
log.Infof("User %d has contributed to Project %d", user.ID, project.ID)

// good
log.Info("New contribution", log.Int("user_id", user.ID), log.Int("project_id", project.ID))
```

**DO NOT** log passwords, emails, credentials, credit cards, personals informations or any other sensitives
informations.

However, log everything that indicates how the application behaves and responds, like seeing a story of the user’s
experience as they used your application.
Also, a profusion of log is better than a lack of them: filter on structured entries are painless.

**Example:**

  * An attempt to authenticate a user was made.
    * Login has succeeded.
    * Login has failed because of an invalid paswword.
    * Login has failed because of an invalid email.
    * etc...
  * A password reset was requested for a user.
  * A password update was requested for a user.
    * Password was updated.
    * Change was discarded because it didn't match.
  * A user has
    * requested a list of available projects.
    * created a new project.
    * deleted a project.
    * uploaded a new media.
    * contributed to a project.
  * A user tried to consult a comment
    * and couldn't read it because he/she didn't have read permissions.
    * and couldn't read it because the comment is no longer available in the database.
    * and successfully read it.
  * A user tried to create a news but an error has occurred
    * because the payload was invalid.
    * because a network error has occurred.
    * because a database error has occurred.

## Context

``context.Context`` should be immutable, don't rely on ``context.Background()`` only to initialize it.

If your context is global don't store too much things in it, keep it simple:

* connection pool (redis, postgresql, rabbitmq, etc.)
* configuration

For the request context include all keys from the application context and add it request information:

* user
* resource for the dedicated endpoint
* user lang

## Error handling

If your method can fail, you need to propagate the error to the root level.

You need to [wrap](https://github.com/pkg/errors) the error and add context.

Always set a recover behavior.

panic/recover is meant for exceptions not common errors.

Always check for errors.

```go
// bad
foo()
val, _ := foo()

// good
_, err := foo()
val, err := foo()
```

Group your logic when checking an error.

```go
// bad
result, err := thisMethodWillFail()

if err != nil {
	return err
}

// bad
if err := thisMethodWillFail(); err != nil {
    return err
}

// good
result, err := thisMethodWillFail()
if err != nil {
	return err
}
```

## Database

 - `Insert` must return created id and date.
 - `Update` must return updated fields and date.
 - `Archive` must return archived date.

Try to avoid using SQL `JOIN`. Use preloads instead.

Use transactions as much as possible. Rollback if an error occured.

Transactions may contain non SQL code like messaging events.

## Functional options

Try to adopt the _functional options_ philosophy when a component requires configurations.

This pattern is divided by three principles:

 1. ComponentOptions
 2. Option
 3. Component

If you are not familiar with it, please read this [introduction](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis).

### ComponentOptions

A `ComponentOptions` is a struct that will configure a `Component` instance.

For example, let's define a `ServerOptions`:

```go
package server

type ServerOptions struct {
   Host string
   Port int
   Logger Logger
}
```

In order to mutate this struct, you'll have to use `Option` function.
_(However, defining default values are allowed)_

### Option

An `Option` is an interface that will configure a `ComponentOptions` using higher-order function.

Because interface can _(and should)_ be public, we have decided to restrict the `Option` visibility so other packages and/or applications can't define new `Option`:

```go
package server

type Option interface {
   apply(*ComponentOptions) error
}

type option func(*ComponentOptions) error

func (o option) apply(options *ComponentOptions) error {
   return o(options)
}
```

With that being said, and with our previous example, we can define `Option` function like this:

```go
func Host(host string) Option {
   return option(func(options *ServerOptions) error {
      options.host = host
      return nil
   })
}

func Port(port int) Option {
   return option(func(options *ServerOptions) error {
      options.port = port
      return nil
   })
}

func WithLogger(logger Logger) Option {
	return option(func(options *ServerOptions) error {
		if logger == nil {
			return errors.New("server: a logger instance is required")
		}
		options.logger = logger
		return nil
	})
}
```

### Component

Once every options are defined, what's left is to implement the `Component` instantiation process.

```go
type Component struct {
    // ...
}

func New(options ...Option) (*Component, error) {
    // ...
}
```

Using our previous example, we should have something like this:

```go
package server

import (
    "net"
    "bytes"
)

type Server struct {
    listener net.Listener
    logger Logger
}

func New(options ...Option) (*Server, error) {

    opts := &ServerOptions{
        host:   "localhost",
        port:   8080,
        logger: &bytes.Buffer{}
    }

    for _, option := range options {
        err := option.apply(opts)
        if err != nil {
            return nil, err
        }
    }

    listener, err := listen(opts.host, opts.port)
    if err != nil {
      return err
    }

    server := &Server {
        listener: listener,
        logger:   opts.logger,
    }

    return server, nil
}
```

## Tests

Don't rely on managers to create your fixtures.

Keep your tests in the same package of your logic. Make them independent, fast and avoid a complex logic.

Keep functional and unit tests separate: you don't need to test a behavior from the HTTP handler.

Prefer writing multiple small tests than an unique integration test which will be slower:

* test your payloads separately
* test your managers using payloads not with an entire http request

Don't mock too much, avoid mocking the main datastore or your application will fail badly.

Use map for multiple checks within the same test.

Table driven tests are great, you should use them:

```go
package foobar_test

func TestFoobar(t *testing.T) {

  scenario := []struct {
    number   int
    expected int
  }{
    {1, 1},
    {2, 2},
    {3, 6},
    {4, 24},
    {5, 120},
    {6, 720},
    {7, 5040},
  }

  for _, tt := range scenario {
    actual := Foobar(tt.number)
    if actual != tt.expected {
      t.Errorf("Foobar(%d): expected %d, received %d", tt.number, tt.expected, actual)
    }
  }

}
```

## Vendor

Sorry you are doomed, we need to find a proper solution for this :)

## Project architecture

This is an initial draft:

```
/application
	/commands
/configuration
	configuration.go
/constants
	constants.go
/events
	users.go
/gimme
	store.go
/failures
	errors.go
	handlers.go
/managers
	users.go
	users_test.go
/models
	user.go
    	user_test.go
/payments
	/backends
/store
    	users.go
    	users_queries.go
/rpc
	/validators
		user.go
	/payloads
		user.go
	/resources
		user.go
/tasks
	users.go
/web
	/authentication
		facebook.go
		ulule.go
	/handlers
		permissions.go
		resources.go
		users.go
	/middleware
		authentication.go
	router.go
	server.go
/worker
	/handlers
		users.go
	worker.go
```

## Naming convention

### Variables

```go
type Category struct{}
type Media struct{}

// slices
mediaList := []Media{}
categories := []Category{}

// maps
categoriesByID := map[int][]Category{}
mediaListByID := map[int][]Media{}
categoryByID := map[int]Category{}
```

### Store methods and managers

```go
func GetCategoryByID(ctx context.Context, id int) (*models.Category, error)
```

```go
func FindCategoriesByID(ctx context.Context, id int) ([]models.Category, error)
```

```go
func FindCategories(ctx context.Context, opts PaginationOptions) ([]models.Category, *Cursor, error)
```

```go
func FindCategoriesByUserID(ctx context.Context, userID int, opts PaginationOptions) ([]models.Category, *Cursor, error)
```

```go
func UpdateCategory(ctx context.Context, category *models.Category) error
```

```go
// hard delete
func DeleteCategory(ctx context.Context, category *models.Category) error
```

```go
// soft delete
func ArchiveCategory(ctx context.Context, category *models.Category) error
```

```go
func CreateCategory(ctx context.Context, category *models.Category) error
```

### Web components

When you need to inject a fully loaded model to your web context, use ``Resource`` suffix.

```go
// resources.go
// inject category in context
func CategoryResource() gin.HandlerFunc
```

Rely on behavior for permissions check.

```go
func CanReadNews() gin.HandlerFunc
```

```go
func CanCreateComments() gin.HandlerFunc
```

```go
func CanCreateProjects() gin.HandlerFunc
```

```go
func CanReadProject() gin.HandlerFunc
```

```go
func CanUpdateProject() gin.HandlerFunc
```

```go
func CanDeleteProject() gin.HandlerFunc
```

```go
func isStaffMember() gin.HandlerFunc
```

```go
func isSuperUser() gin.HandlerFunc
```

```go
func isAuthenticated() gin.HandlerFunc
```

```go
func isAnonymous() gin.HandlerFunc
```

### Tests

Name your tests as follows: `Test<Package>_<TestedFunc>`.

Example: `TestViews_OrderDetail` means we are testing the `OrderDetail` func from `views` package.

To run all `Order` related tests you can launch: `go test -run Order`
