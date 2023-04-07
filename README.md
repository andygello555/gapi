# gapi

An agnostic wrapper for creating Go bindings to all your favourite web APIs.

## Why

Have you ever ran into these problems when developing a Go project that uses obscure/new web APIs?

1. You want to use a web API, but are no Go clients/bindings/packages available for it
2. Maybe there is one, but it is large/complex, and you only really need a subset of its functionality

Then **gapi** is for you!

## What does it do

1. Makes it easy to create bindings for actions within that web API
2. Makes it easy to use paginated API actions, **even with rate limits!**
3. Type checked arguments can be passed to every binding that you create which will cause appropriate errors at runtime
4. The entire request pipeline for a binding can be easily modified at any point
5. Supports Go generics so that you can create bindings that will return type-checked values at compile time!

## Example API

The following example defines an API client + bindings for the [Fake Store REST API](https://fakestoreapi.com/).

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"github.com/andygello555/agem"
	"github.com/andygello555/gapi"
	"github.com/pkg/errors"
	"io"
	"net/http"
	"net/url"
	"strconv"
)

// Product defines a product returned by the fakestoreapi.
type Product struct {
	ID          int     `json:"id"`
	Title       string  `json:"title"`
	Price       float64 `json:"price"`
	Category    string  `json:"category"`
	Description string  `json:"description"`
	Image       string  `json:"image"`
}

// User defines a user returned by the fakestoreapi.
type User struct {
	ID       int    `json:"id"`
	Email    string `json:"email"`
	Username string `json:"username"`
	Password string `json:"password"`
	Name     struct {
		Firstname string `json:"firstname"`
		Lastname  string `json:"lastname"`
	} `json:"name"`
	Address struct {
		City        string `json:"city"`
		Street      string `json:"street"`
		Number      int    `json:"number"`
		Zipcode     string `json:"zipcode"`
		Geolocation struct {
			Lat  string `json:"lat"`
			Long string `json:"long"`
		} `json:"geolocation"`
	} `json:"address"`
	Phone string `json:"phone"`
}

type httpClient struct {
	*http.Client
}

func (h httpClient) Run(ctx context.Context, bindingName string, attrs map[string]any, req api.Request, res any) (err error) {
	request := req.(api.HTTPRequest).Request

	var response *http.Response
	if response, err = h.Do(request); err != nil {
		return err
	}

	if response.Body != nil {
		defer func(body io.ReadCloser) {
			err = agem.MergeErrors(err, errors.Wrapf(body.Close(), "could not close response body to %s", request.URL.String()))
		}(response.Body)
	}

	var body []byte
	if body, err = io.ReadAll(response.Body); err != nil {
		err = errors.Wrapf(err, "could not read response body to %s", request.URL.String())
		return
	}

	err = json.Unmarshal(body, res)
	return
}

func main() {
	// Then we create a Client instance. Here httpClient is a type that implements the Client interface, where
	// Client.Run performs an HTTP request using http.DefaultClient, and then unmarshals the JSON response into the
	// response wrapper.
	client := httpClient{http.DefaultClient}

	// Finally, we create the API itself by creating and registering all our Bindings within the Schema using the
	// NewWrappedBinding method. The "users" and "products" Bindings take only one argument: the limit argument. This
	// limits the number of resources returned by the fakestoreapi. This is applied to the Request by setting the query
	// params for the http.Request.
	a := api.NewAPI(client, api.Schema{
		// Note: we do not supply a wrap and an unwrap method for the "users" and "products" Bindings because the
		//       fakestoreapi returns JSON that can be unmarshalled straight into an appropriate instance of type ResT.
		//       We also don't need to supply a response method because the ResT type is the same as the RetT type.
		"users": api.NewWrappedBinding("users",
			func(b api.Binding[[]User, []User], args ...any) (request api.Request) {
				u, _ := url.Parse("https://fakestoreapi.com/users")
				if len(args) > 0 {
					query := u.Query()
					query.Add("limit", strconv.Itoa(args[0].(int)))
					u.RawQuery = query.Encode()
				}
				req, _ := http.NewRequest(http.MethodGet, u.String(), nil)
				return api.HTTPRequest{req}
			}, nil, nil, nil,
			func(binding api.Binding[[]User, []User]) []api.BindingParam {
				return api.Params("limit", 1)
			}, false,
		),
		"products": api.NewWrappedBinding("products",
			func(b api.Binding[[]Product, []Product], args ...any) api.Request {
				u, _ := url.Parse("https://fakestoreapi.com/products")
				if len(args) > 0 {
					query := u.Query()
					query.Add("limit", strconv.Itoa(args[0].(int)))
					u.RawQuery = query.Encode()
				}
				req, _ := http.NewRequest(http.MethodGet, u.String(), nil)
				return api.HTTPRequest{req}
			}, nil, nil, nil,
			func(binding api.Binding[[]Product, []Product]) []api.BindingParam {
				return api.Params("limit", 1)
			}, false,
		),
		// The "first_product" Binding showcases how to set the response method, as well as how to use the chaining API
		// when creating Bindings. This will execute a similar HTTP request to the "products" Binding but
		// Binding.Execute will instead return a single Product instance.
		// Note: how the RetT type param is set to just "Product".
		"first_product": api.WrapBinding(api.NewBindingChain(func(binding api.Binding[[]Product, Product], args ...any) (request api.Request) {
			req, _ := http.NewRequest(http.MethodGet, "https://fakestoreapi.com/products?limit=1", nil)
			return api.HTTPRequest{req}
		}).SetResponseMethod(func(binding api.Binding[[]Product, Product], response []Product, args ...any) Product {
			return response[0]
		}).SetName("first_product")),
	})

	// Then we can execute our "users" binding with a limit of 3...
	var resp any
	var err error
	if resp, err = a.Execute("users", 3); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(resp.([]User))

	// ...and we can also execute our "products" binding with a limit of 1...
	if resp, err = a.Execute("products", 1); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(resp.([]Product))

	// ...and we can also execute our "first_product" binding.
	if resp, err = a.Execute("first_product"); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(resp.(Product))

	// Finally, we will check whether the "limit" parameter for the "users" action defaults to 1
	if resp, err = a.Execute("users"); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(resp.([]User))
}
```
