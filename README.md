# README

This codebase has been generated by [Autostrada](https://autostrada.dev/).

## Getting started

Before running the application you will need a working PostgreSQL installation and a valid DSN (data source name) for connecting to the database.

Please open the `cmd/web/main.go` file and edit it to include your valid DSN as the default value.

```
cfg.db.dsn = env.GetString("DB_DSN", "YOUR DEFAULT DSN GOES HERE")
```

Note that this DSN must be in the format `user:pass@localhost:port/db` and **not** be prefixed with `postgres://`.

Make sure that you're in the root of the project directory, fetch the dependencies with `go mod tidy`, then run the application using `go run ./cmd/web`:

```
$ go mod tidy
$ go run ./cmd/web
```

Then visit [http://localhost:4444](http://localhost:4444) in your browser.

## Project structure

Everything in the codebase is designed to be editable. Feel free to change and adapt it to meet your needs.

|     |     |
| --- | --- |
| **`assets`** | Contains the non-code assets for the application. |
| `↳ assets/static/` | Contains static UI files (images, CSS etc). |
| `↳ assets/templates/` | Contains HTML templates. |
| `↳ assets/efs.go` | Declares an embedded filesystem containing all the assets. |

|     |     |
| --- | --- |
| **`cmd/web`** | Your application-specific code (handlers, routing, middleware, helpers) for dealing with HTTP requests and responses. |
| `↳ cmd/web/errors.go` | Contains helpers for managing and responding to error conditions. |
| `↳ cmd/web/handlers.go` | Contains your application HTTP handlers. |
| `↳ cmd/web/helpers.go` | Contains helper functions for common tasks. |
| `↳ cmd/web/main.go` | The entry point for the application. Responsible for parsing configuration settings initializing dependencies and running the server. Start here when you're looking through the code. |
| `↳ cmd/web/middleware.go` | Contains your application middleware. |
| `↳ cmd/web/routes.go` | Contains your application route mappings. |
| `↳ cmd/web/server.go` | Contains a helper functions for starting and gracefully shutting down the server. |

|     |     |
| --- | --- |
| **`internal`** | Contains various helper packages used by the application. |
| `↳ internal/database/` | Contains your database-related code (setup, connection and queries). |
| `↳ internal/env` | Contains helper functions for reading configuration settings from environment variables. |
| `↳ internal/funcs/` | Contains custom template functions. |
| `↳ internal/request/` | Contains helper functions for decoding HTML forms, JSON requests, and URL query strings. |
| `↳ internal/response/` | Contains helper functions for rendering HTML templates and sending JSON responses. |
| `↳ internal/validator/` | Contains validation helpers. |
| `↳ internal/version/` | Contains the application version number definition. |

## Configuration settings

Configuration settings are managed via environment variables, with the environment variables read into your application in the `run()` function in the `main.go` file.

You can try this out by setting a `HTTP_PORT` environment variable to configure the network port that the server is listening on:

```
$ export HTTP_PORT="9999"
$ go run ./cmd/web
```

Feel free to adapt the `run()` function to parse additional environment variables and store their values in the `config` struct. The application uses helper functions in the `internal/env` package to parse environment variable values or return a default value if no matching environment variable is set. It includes `env.GetString()`, `env.GetInt()` and `env.GetBool()` functions for reading string, integer and bool values from environment variables. Again, you can add any additional helper functions that you need.

## Creating new handlers

Handlers are defined as `http.HandlerFunc` methods on the `application` struct. They take the pattern:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    // Your handler logic...
}
```

Handlers are defined in the `cmd/web/handlers.go` file. For small applications, it's fine for all handlers to live in this file. For larger applications (10+ handlers) you may wish to break them out into separate files.

## Handler dependencies

Any dependencies that your handlers have should be initialized in the `run()` function `cmd/web/main.go` and added to the `application` struct. All of your handlers, helpers and middleware that are defined as methods on `application` will then have access to them.

You can see an example of this in the `cmd/web/main.go` file where we initialize a new `logger` instance and add it to the `application` struct.

## Creating new routes

[HttpRouter](https://github.com/julienschmidt/httprouter) is used for routing. Routes are defined in the `routes()` method in the `cmd/web/routes.go` file. For example:

```
func (app *application) routes() http.Handler {
    mux := httprouter.New()

    mux.HandlerFunc("GET", "/your/path", app.yourHandler)

    return mux
}
```

For more information about HttpRouter and example usage, please see the [official documentation](https://github.com/julienschmidt/httprouter).

## Adding middleware

Middleware is defined as methods on the `application` struct in the `cmd/web/middleware.go` file. Feel free to add your own. They take the pattern:

```
func (app *application) yourMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Your middleware logic...
        next.ServeHTTP(w, r)
    })
}
```

You can then use this middleware by wrapping the router before returning it from the `routes()` method, like so:

```
func (app *application) routes() http.Handler {
    mux := httprouter.New()

    mux.HandlerFunc("GET", "/your/path", app.yourHandler)

    // Wrap the router with middleware.
    return app.yourMiddlware(app.yourOtherMiddleware(mux))
}
```

It's possible to use middleware on specific routes only:

```
func (app *application) routes() http.Handler {
    mux := httprouter.New()

    mux.HandlerFunc("GET", "/your/path", app.yourHandler)

    // Wrap this handler with route-specific middleware. Note that when
    // wrapping handler functions with route-specific middleware that you
    // need to convert them to a http.Handler by using the http.HandlerFunc()
    // adapter. Like so:
    mux.Handler("GET", "/your/other/path", app.yourOtherMiddleware(http.HandlerFunc(app.yourOtherHandler)))

    return app.yourMiddleware(mux)
}
```

## Rendering HTML templates

HTML templates are stored in the `assets/templates` directory and use the standard library `html/template` package. The structure looks like this:

|     |     |
| --- | --- |
| `assets/templates/base.tmpl` | The 'base' template containing the shared HTML markup for all your web pages. |
| `assets/templates/pages/` | Directory containing files with the page-specific content for your web pages. See `assets/templates/pages/home.tmpl` for an example. |
| `assets/templates/partials/` | Directory containing files with 'partials' to embed in your web pages or base template. See `assets/templates/partials/footer.tmpl` for an example. |

The HTML for web pages can be sent using the `response.Page()` function. For convenience, an `app.newTemplateData()` method is provided which returns a `map[string]any` map. You can add data to this map and pass it on to your templates.

For example, to render the HTML in a `assets/templates/pages/example.tmpl` file:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    data := app.newTemplateData()
    data["hello"] = "world"

    err := response.Page(w, http.StatusOK, data, "pages/example.tmpl")
    if err != nil {
        app.serverError(w, r, err)
    }
}
```

Specific HTTP headers can optionally be sent with the response too:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    data := app.newTemplateData()
    data["hello"] = "world"

    headers := make(http.Header)
    headers.Set("X-Server", "Go")

    err := response.PageWithHeaders(w, http.StatusOK, data, headers, "pages/example.tmpl")
    if err != nil {
        app.serverError(w, r, err)
    }
}
```

Note: All the files in the `assets/templates` directory are embedded into your application binary and can be accessed via the `EmbeddedFiles` variable in `assets/efs.go`.

## Adding default template data

If you have data that you want to display or use on multiple web pages, you can adapt the `newTemplateData()` helper in the `helpers.go` file to include this by default. For example, if you wanted to include the current year value you could adapt it like this:

```
func (app *application) newTemplateData() map[string]any {
    data := map[string]any{
        "CurrentYear": time.Now().Year(),
    }

    return data
}
```

## Custom template functions

Custom template functions are defined in `internal/funcs/funcs.go` and are automatically made available to your

HTML templates when you use `response.Page()`
.

The following custom template functions are already included by default:

|     |     |
| --- | --- |
| `now` | Returns the current time. |
| `timeSince arg1` | Returns the time elapsed since arg1. |
| `timeUntil arg2` | Returns the time until arg1. |
| `formatTime arg1 arg2` | Returns the time arg2 as formatted using the pattern arg1. |
| `approxDuration arg1` | Returns the approximate duration of arg1 in a 'human-friendly' format ("3 seconds", "2 months", "5 years") etc. |
| `uppercase arg1` | Returns arg1 converted to uppercase. |
| `lowercase arg1` | Returns arg1 converted to lowercase. |
| `pluralize arg1 arg2 arg3` | If arg1 equals 1 then return arg2, otherwise return arg3. |
| `slugify arg1` | Returns the lowercase of arg1 with all non-ASCII characters and punctuation removed (expect underscores and hyphens). Whitespaces are also replaced with a hyphen. |
| `safeHTML arg1` | Output the verbatim value of arg1 without escaping the content. This should only be used when arg1 is from a trusted source. |
| `join arg1 arg2` | Returns the values in slice arg1 joined using the separator arg2. |
| `incr arg1` | Increments arg1 by 1. |
| `decr arg1` | Decrements arg1 by 1. |
| `formatInt arg1` | Returns arg1 formatted with commas as the thousands separator. |
| `formatFloat arg1 arg2` | Returns arg1 rounded to arg2 decimal places and formatted with commas as the thousands separator. |
| `yesno arg1` | Returns "Yes" if arg1 is true, or "No" if arg1 is false. |
| `urlSetParam arg1 arg2 arg3` | Returns the URL arg1 with the key arg2 and value arg3 added to the query string parameters. |
| `urlDelParam arg1 arg2` | Returns the URL arg1 with the key arg2 (and corresponding value) removed from the query string parameters. |

To add another custom template function, define the function in `internal/funcs/funcs.go` and add it to the `TemplateFuncs` map. For example:

```
var TemplateFuncs = template.FuncMap{
    ...
    "yourFunction": yourFunction,
}

func yourFunction(s string) (string, error) {
    // Do something...
}
```

## Static files

By default, the files in the `assets/static` directory are served using Go's `http.Fileserver` whenever the application receives a `GET` request with a path beginning `/static/`. So, for example, if the application receives a `GET /static/css/main.css` request it will respond with the contents of the `assets/static/css/main.css` file.

If you want to change or remove this behavior you can by editing the `routes.go` file.

Note: The files in `assets/static` directory are embedded into your application binary and can be accessed via the `EmbeddedFiles` variable in `assets/efs.go`.

## Working with forms

The codebase includes a `request.DecodePostForm()` function for automatically decoding HTML form data from a POST request into a struct, and `request.DecodeQueryString()` for decoding URL query strings into a struct. You can also use the `request.DecodeForm()` function to decode both form data from a POST request and query string data at the same time. Behind the scenes decoding is managed using the [go-playground/form](https://github.com/go-playground/form) package.

As an example, let's say you have a page with the following HTML form for creating a 'person' record and routing rule:

```
<form action="/person/create" method="POST">
    <div>
        <label>Your name:</label>
        <input type="text" name="Name" value="{{.Form.Name}}">
    </div>
    <div>
        <label>Your age:</label>
        <input type="number" name="Age" value="{{.Form.Age}}">
    </div>
    <button>Submit</button>
</form>
```

```
func (app *application) routes() http.Handler {
    mux := flow.New()

    mux.HandleFunc("/person/create", app.createPerson, "GET", "POST")

    return mux
}
```

Then you can display and parse this form with a `createPerson` handler like this:

```
package main

import (
    "net/http"

    "example.com/internal/request"
    "example.com/internal/response"
)

func (app *application) createPerson(w http.ResponseWriter, r *http.Request) {
    type createPersonForm struct {
        Name string `form:"Name"`
        Age  int    `form:"Age"`
    }

    switch r.Method {
    case http.MethodGet:
        data := app.newTemplateData()

        // Add any default values to the form.
        data["Form"] = createPersonForm{
            Age: 21,
        }

        err := response.Page(w, http.StatusOK, data, "/path/to/page.tmpl")
        if err != nil {
            app.serverError(w, r, err)
        }

    case http.MethodPost:
        var form createPersonForm

        err := request.DecodePostForm(r, &form)
        if err != nil {
            app.badRequest(w, r, err)
            return
        }

        // Do something with the data in the form variable...
    }
}
```

## Validating forms

The `internal/validator` package includes a simple (but powerful) `validator.Validator` type that you can use to carry out validation checks.

Extending the example above:

```
package main

import (
    "net/http"

    "example.com/internal/request"
    "example.com/internal/response"
    "example.com/internal/validator"
)

func (app *application) createPerson(w http.ResponseWriter, r *http.Request) {
    type createPersonForm struct {
        Name      string              `form:"Name"`
        Age       int                 `form:"Age"`
        Validator validator.Validator `form:"-"`
    }

    switch r.Method {
    case http.MethodGet:
        data := app.newTemplateData()

        // Add any default values to the form.
        data["Form"] = createPersonForm{
            Age: 21,
        }

        err := response.Page(w, http.StatusOK, data, "/path/to/page.tmpl")
        if err != nil {
            app.serverError(w, r, err)
        }

    case http.MethodPost:
        var form createPersonForm

        err := request.DecodePostForm(r, &form)
        if err != nil {
            app.badRequest(w, r, err)
            return
        }

        form.Validator.CheckField(form.Name != "", "Name", "Name is required")
        form.Validator.CheckField(form.Age != 0, "Age", "Age is required")
        form.Validator.CheckField(form.Age >= 21, "Age", "Age must be 21 or over")

        if form.Validator.HasErrors() {
            data := app.newTemplateData()
            data["Form"] = form

            err := response.Page(w, http.StatusUnprocessableEntity, data, "/path/to/page.tmpl")
            if err != nil {
                app.serverError(w, r, err)
            }
            return
        }

        // Do something with the form information, like adding it to a database...
    }
}
```

And you can display the error messages in your HTML form like this:

```
<form action="/person/create" method="POST">
    {{if .Form.Validator.HasErrors}}
        <p>Something was wrong. Please correct the errors below and try again.</p>
    {{end}}
    <div>
        <label>Your name:</label>
        {{with .Form.Validator.FieldErrors.Name}}
            <span class='error'>{{.}}</span>
        {{end}}
        <input type="text" name="Name" value="{{.Form.Name}}">
    </div>
    <div>
        <label>Your age:</label>
        {{with .Form.Validator.FieldErrors.Age}}
            <span class='error'>{{.}}</span>
        {{end}}
        <input type="number" name="Age" value="{{.Form.Age}}">
    </div>
    <button>Submit</button>
</form>
```

In the example above we use the `CheckField()` method to carry out validation checks for specific fields. You can also use the `Check()` method to carry out a validation check that is _not related to a specific field_. For example:

```
input.Validator.Check(input.Password == input.ConfirmPassword, "Passwords do not match")
```

The `validator.AddError()` and `validator.AddFieldError()` methods also let you add validation errors directly:

```
input.Validator.AddFieldError("Email", "This email address is already taken")
input.Validator.AddError("Passwords do not match")
```

The `internal/validator/helpers.go` file also contains some helper functions to simplify validations that are not simple comparison operations.

|     |     |
| --- | --- |
| `NotBlank(value string)` | Check that the value contains at least one non-whitespace character. |
| `MinRunes(value string, n int)` | Check that the value contains at least n runes. |
| `MaxRunes(value string, n int)` | Check that the value contains no more than n runes. |
| `Between(value, min, max T)` | Check that the value is between the min and max values inclusive. |
| `Matches(value string, rx *regexp.Regexp)` | Check that the value matches a specific regular expression. |
| `In(value T, safelist ...T)` | Check that a value is in a 'safelist' of specific values. |
| `AllIn(values []T, safelist ...T)` | Check that all values in a slice are in a 'safelist' of specific values. |
| `NotIn(value T, blocklist ...T)` | Check that the value is not in a 'blocklist' of specific values. |
| `NoDuplicates(values []T)` | Check that a slice does not contain any duplicate (repeated) values. |
| `IsEmail(value string)` | Check that the value has the formatting of a valid email address. |
| `IsURL(value string)` | Check that the value has the formatting of a valid URL. |

For example, to use the `Between` check your code would look similar to this:

```
input.Validator.CheckField(validator.Between(input.Age, 18, 30), "Age", "Age must between 18 and 30")
```

Feel free to add your own helper functions to the `internal/validator/helpers.go` file as necessary for your application.

## Sending JSON responses

JSON responses and a specific HTTP status code can be sent using the `response.JSON()` function. The `data` parameter can be any JSON-marshalable type.

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    data := map[string]string{"hello":  "world"}

    err := response.JSON(w, http.StatusOK, data)
    if err != nil {
        app.serverError(w, r, err)
    }
}
```

Specific HTTP headers can optionally be sent with the response too:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    data := map[string]string{"hello":  "world"}

    headers := make(http.Header)
    headers.Set("X-Server", "Go")

    err := response.JSONWithHeaders(w, http.StatusOK, data, headers)
    if err != nil {
        app.serverError(w, r, err)
    }
}
```

## Parsing JSON requests

HTTP requests containing a JSON body can be decoded using the `request.DecodeJSON()` function. For example, to decode JSON into an `input` struct:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Name string `json:"Name"`
        Age  int    `json:"Age"`
    }

    err := request.DecodeJSON(w, r, &input)
    if err != nil {
        app.badRequest(w, r, err)
        return
    }

    ...
}
```

Note: The target decode destination passed to `request.DecodeJSON()` (which in the example above is `&input`) must be a non-nil pointer.

The `request.DecodeJSON()` function returns friendly, well-formed, error messages that are suitable to be sent directly to the client using the `app.badRequest()` helper.

There is also a `request.DecodeJSONStrict()` function, which works in the same way as `request.DecodeJSON()` except it will return an error if the request contains any JSON fields that do not match a name in the the target decode destination.

## Validating JSON requests

The `internal/validator` package includes a simple (but powerful) `validator.Validator` type that you can use to carry out validation checks.

Extending the example above:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Name      string              `json:"Name"`
        Age       int                 `json:"Age"`
        Validator validator.Validator `json:"-"`
    }

    err := request.DecodeJSON(w, r, &input)
    if err != nil {
        app.badRequest(w, r, err)
        return
    }

    input.Validator.CheckField(input.Name != "", "Name", "Name is required")
    input.Validator.CheckField(input.Age != 0, "Age", "Age is required")
    input.Validator.CheckField(input.Age >= 21, "Age", "Age must be 21 or over")

    if input.Validator.HasErrors() {
        app.failedValidation(w, r, input.Validator)
        return
    }

    ...
}
```

The `app.failedValidation()` helper will send a `422` status code along with any validation error messages. For the example above, the JSON response will look like this:

```
{
    "FieldErrors": {
        "Age": "Age must be 21 or over",
        "Name": "Name is required"
    }
}
```

## Working with the database

This codebase is set up to use PostgreSQL with the [lib/pq](https://github.com/lib/pq) driver. You can control which database you connect to using the `DB_DSN` environment variable to pass in a DSN, or by adapting the default value in `run()`.

The codebase is also configured to use [jmoiron/sqlx](https://github.com/jmoiron/sqlx), so you have access to the whole range of sqlx extensions as well as the standard library `Exec()`, `Query()` and `QueryRow()` methods .

The database is available to your handlers, middleware and helpers via the `application` struct. If you want, you can access the database and carry out queries directly. For example:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    ...

    _, err := app.db.Exec("INSERT INTO people (name, age) VALUES ($1, $2)", "Alice", 28)
    if err != nil {
        app.serverError(w, r, err)
        return
    }

    ...
}
```

Generally though, it's recommended to isolate your database logic in the `internal/database` package and extend the `DB` type to include your own methods. For example, you could create a `internal/database/people.go` file containing code like:

```
type Person struct {
    ID    int    `db:"id"`
    Name  string `db:"name"`
    Age   int    `db:"age"`
}

func (db *DB) NewPerson(name string, age int) error {
    _, err := db.Exec("INSERT INTO people (name, age) VALUES ($1, $2)", name, age)
    return err
}

func (db *DB) GetPerson(id int) (Person, error) {
    var person Person
    err := db.Get(&person, "SELECT * FROM people WHERE id = $1", id)
    return person, err
}
```

And then call this from your handlers:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    ...

    err := app.db.NewPerson("Alice", 28)
    if err != nil {
        app.serverError(w, r, err)
        return
    }

    ...
}
```

## Logging

Leveled logging is supported using the [slog](https://pkg.go.dev/log/slog) package.

By default, a logger is initialized in the `main()` function. This logger writes all log messages above `Debug` level to `os.Stdout`.

```
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug}))
```

Feel free to customize this further as necessary.

Also note: Any messages that are automatically logged by the Go `http.Server` are output at the `Warn` level.

## Using Basic Authentication

The `cmd/web/middleware.go` file contains a `basicAuth` middleware that you can use to protect your application — or specific application routes — with HTTP basic authentication.

You can try this out by visiting the [https://localhost:4444/basic-auth-protected](https://localhost:4444/basic-auth-protected) endpoint in any web browser and entering the default user name and password:

```
User name: admin
Password:  pa55word
```

You can change the user name and password by setting the `BASIC_AUTH_USERNAME` environment variable and `BASIC_AUTH_HASHED_PASSWORD` environment variable. For example:

```
$ export BASIC_AUTH_USERNAME='alice'
$ export BASIC_AUTH_HASHED_PASSWORD='$2a$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
$ go run ./cmd/web
```

Note: You will probably need to wrap the username and password in `'` quotes to prevent your shell interpreting dollar and slash symbols as special characters.

The value for the `BASIC_AUTH_HASHED_PASSWORD` environment variable should be a bcrypt hash of the password, not the plaintext password itself. An easy way to generate the bcrypt hash for a password is to use the `gophers.dev/cmds/bcrypt-tool` package like so:

```
$ go run gophers.dev/cmds/bcrypt-tool@latest hash 'your_pa55word'
```

If you want to change the default values for username and password you can do so by editing the default command-line flag values in the `cmd/web/main.go` file.

## Admin tasks

The `Makefile` in the project root contains commands to easily run common admin tasks:

|     |     |
| --- | --- |
| `$ make tidy` | Format all code using `go fmt` and tidy the `go.mod` file. |
| `$ make audit` | Run `go vet`, `staticheck`, `govulncheck`, execute all tests and verify required modules. |
| `$ make test` | Run all tests. |
| `$ make test/cover` | Run all tests and outputs a coverage report in HTML format. |
| `$ make build` | Build a binary for the `cmd/web` application and store it in the `/tmp/bin` folder. |
| `$ make run` | Build and then run a binary for the `cmd/web` application. |

## Running background tasks

A `backgroundTask()` helper is included in the `cmd/web/helpers.go` file. You can call this in your handlers, helpers and middleware to run any logic in a separate background goroutine. This useful for things like sending emails, or completing slow-running jobs.

You can call it like so:

```
func (app *application) yourHandler(w http.ResponseWriter, r *http.Request) {
    ...

    app.backgroundTask(r, func() error {
        // The logic you want to execute in a background task goes here.
        // It should return an error, or nil.
        err := doSomething()
        if err != nil {
            return err
        }

        return nil
    })

    ...
}
```

Using the `backgroundTask()` helper will automatically recover any panics in the background task logic, and when performing a graceful shutdown the application will wait for any background tasks to finish running before it exits.

## Application version

The application version number is defined in a `Get()` function in the `internal/version/version.go` file. Feel free to change this as necessary.

```
package version

func Get() string {
    return "0.0.1"
}
```

## Changing the module path

The module path is currently set to `example.com`. If you want to change this please find and replace all instances of `example.com` in the codebase with your own module path.