# Go Style Guide

*A mostly reasonable approach to Go*

## Wisdom Quotes

KISS

Don't repeat yourself

Stay [idiomatic](https://golang.org/doc/effective_go.html)

Embrace [12factor](https://12factor.net/) as much as possible

## Package names

Avoid too much level for your packages (3 levels is mostly enough).

Prefer package names based on feature instead of which software they are interacting to.

For example:

* ``redis`` will become ``caching``
* ``rabbitmq`` will become ``brokers`` or ``messaging``

For these kind of packages, you will include a ``dummy`` implementation to facilitate unittests.

An example of ``rabbitmq`` dummy implementation will use a in-memory storage to store events.

## Required development tools

* [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports)
* [golint](https://github.com/golang/lint)
* [gometalinter](https://github.com/alecthomas/gometalinter) which includes previous tools and run them asynchronously
* [gofmt](https://golang.org/cmd/gofmt/) with the ``-s`` option to simplify code

Integrate those tools with your own editor.

## Be readable

Give a short but explicit name to your vars, functions and use ``lowerCamelCase`` representation.

```golang
// bad
a := "alx@ulule.com"
userEmailWithRootPermissions := "flo@ulule.com"

// good
userEmail := "gilles@ulule.com"
email := "louise@ulule.com"
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

1. Stdlib
2. External packages
3. Internal packages

In common case, if you need to rename packages you are doing it wrong.

## Be secure

Always check users input by using validators and sanitize them.

Don't sanitize an input which is already validating.

There is no trusted sources.

Do not assume, verify your output.

Always ``seed(time.Now().Unix())`` before using [rand](https://golang.org/pkg/math/rand/).

## Be realistic

Your code will fail, use [recover](https://blog.golang.org/defer-panic-and-recover), sentry, alerting, etc.

Internal / external services will also fail, rely on services degradation as much as possible.

## Code organization

Be explicit when naming thing.

```golang
// bad
package user

func Get()
```

```golang
// good
package models

func GetUser()
```

Avoid doing useless tests when your method return a ``bool`` or the same output of a calling method.

```golang
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
```

Group your variable declarations.

```golang

// bad
users := map[int]*User{}
exists := false

// good
var (
	users = map[int]*User{}
	exists = false
)
```

Rely on [context](https://golang.org/pkg/context/) everywhere.

## Error handling

If your method can fail, you need to propagate the error the root level.

Always set a recover.

panic/recover is meant for exceptions not common errors.

Always check for errors.

```golang
// bad
foo()
val, _ := foo()

// good
_, err := foo()
val, err := foo()
```

Group your logic when checking an error.

```golang
// bad
result, err := thisMethodWillFail()

if err != nil {
	return err
}

// good
result, err := thisMethodWillFail()
if err != nil {
	return err
}
```

## Tests

Keep your tests in the same package of your logic.

Make them independants, fast and avoid a complex logic.

Keep functional and unit tests separate: you don't need to test a behavior from the HTTP handler.

Don't mock too much, avoid mocking the main datastore or your tests will fail badly.

Use map for multiples check within the same test.

## Vendor

Sorry you are doomed, we need to find a proper solution for this :)

## Project architecture

This is an initial draft:

```
/application
	/commands
/managers
/constants
/payments
	/backends
/events
/stores
	/postgresql
		/models
		/queries
	/cassandra
		/models
		/queries
/api
	/payloads
	/resources
/web
	server.go
	/handlers
		permissions.go
		resources.go
		foo.go
	/middlewares
		auth.go
	/auth
		facebook.go
		ulule.go
/worker
	/tasks
	/handlers
	worker.go
```

## Naming convention

```golang
type Category struct {}

// slices
categoryList := []Category{}
categories := []Category{}

// maps
categoriesByID := map[int][]Category{}
categoryByID := map[int]Category{}
```

For store and manager methods:

```golang
func GetCategoryByID(ctx context.Context, id int) (Category, error)
```

```golang
func FindCategoriesByID(ctx context.Context, id int) ([]Category, error)
```

```golang
func FindCategories(ctx context.Context, opts PaginationOptions) ([]models.Category, *Cursor, error)
```

```golang
func FindCategoriesByUserId(ctx context.Context, userId int, opts PaginationOptions) ([]models.Category, *Cursor, error)
```

```golang
func UpdateCategory(ctx context.Context, category *models.Category) error
```

```golang
// hard delete
func DeleteCategory(ctx context.Context, category *models.Category) error
```

```golang
// soft delete
func ArchiveCategory(ctx context.Context, category *models.Category) error
```

```golang
func CreateCategory(ctx context.Context, category *models.Category) error
```

When you need to inject a fully loaded model to your web context, use ``Resource`` suffix.

```golang
// resources.go
// inject category in context
func CategoryResource() gin.HandlerFunc
```

Rely on behavior for permissions check.

```golang
func CanReadNews() gin.HandlerFunc
```

```golang
func CanCreateComments() gin.HandlerFunc
```

```golang
func CanCreateProjects() gin.HandlerFunc
```

```golang
func CanReadProject() gin.HandlerFunc
```

```golang
func CanUpdateProject() gin.HandlerFunc
```

```golang
func CanDeleteProject() gin.HandlerFunc
```

```golang
func isStaffMember() gin.HandlerFunc
```

```golang
func isSuperUser() gin.HandlerFunc
```

```golang
func isAuthenticated() gin.HandlerFunc
```

```golang
func isAnonymous() gin.HandlerFunc
```
