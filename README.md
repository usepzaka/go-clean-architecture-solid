# GO Clean Architecture with Solid Principals

![Gopher](https://pkg.go.dev/static/shared/gopher/package-search-700x300.jpeg)

**Go project** following **Clean Architecture**, **SOLID principles**, **Dependency Injection (DI)**, **GORM** (for database interaction), and implementing **System Tests** and **Mocking** with **Go Fiber**. 
We structure the project in a way that makes the **domain layer** (entities and models) reusable by other services. It can be use to make monolith with modularity.

## The project will follow the key guidelines:

1.  **Clean Architecture**: Clear separation of concerns.
    
2.  **SOLID principles**: Maintain high cohesion, low coupling, and proper interfaces.
    
3.  **GORM**: To handle database interaction in the **infrastructure layer**.
    
4.  **Go Fiber**: Web framework for handling HTTP requests.
    
5.  **Dependency Injection (DI)**: To make components decoupled and easily testable.
    
6.  **Reusability**: The domain layer (entities and models) is shared across different services.
   


## This project has 4 Domain layer :

-   Domain/Model Layer
-   Repository Layer
-   Usecase/Service Layer
-   Controller/Delivery Layer

## The diagram:
   
![Clean Architecture](images/clean-arch.png)



### 1. Project Structure
```text
go-clean-architecture/
│
├── app/
│   ├── user/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── user.go
│   ├── product/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── product.go
│   ├── payment/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── payment.go
│   ├── main.go
│   └── server.go
│
├── domain/
│   ├── user/
│   │   └── model.go
│   ├── product/
│   │   └── model.go
│   ├── payment/
│   │   └── model.go
│   └── common/
│       ├── error.go
│       └── logger.go
│
├── migrations/
│   ├── migrations.go
│   └── seed.go
│
├── config/
│   └── config.go
│
├── scripts/
│   └── init.sh
│
└── go.mod

```

### 2. Dependencies

In your `go.mod`, you need to import the required libraries:

```go
module go-clean-architecture

go 1.18

require (
    github.com/gofiber/fiber/v2 v2.36.0
    github.com/jinzhu/gorm v1.9.16
    github.com/jinzhu/gorm/dialects/sqlite
    github.com/stretchr/testify v1.8.1
    github.com/go-stack/stack v1.8.0
)
```

### 3. Main Application (main.go & server.go)
 `main.go`

```go
// main.go
package main

import (
	"log"
	"github.com/gofiber/fiber/v2"
	"go-clean-architecture/app/user"
	"go-clean-architecture/app/product"
	"go-clean-architecture/app/payment"
	"go-clean-architecture/config"
)

func main() {
	// Initialize configuration
	cfg := config.LoadConfig()

	// Initialize Database Connection
	db := config.SetupDatabase(cfg)

	// Initialize the app
	app := fiber.New()

	// Register services
	userService := user.NewService(db)
	productService := product.NewService(db)
	paymentService := payment.NewService(db)

	// Register routes (Handlers)
	user.RegisterRoutes(app, userService)
	product.RegisterRoutes(app, productService)
	payment.RegisterRoutes(app, paymentService)

	// Start server
	log.Fatal(app.Listen(":3000"))
}
```

`server.go`

```go
// server.go
package main

import (
	"github.com/gofiber/fiber/v2"
)

func SetupRoutes(app *fiber.App) {
	// Setup routes for different services
}
```


### 4. Config (config/config.go)

```go
package config

import (
	"log"
	"github.com/jinzhu/gorm"
	"github.com/gofiber/fiber/v2"
	"os"
)

type Config struct {
	DatabaseURL string
}

func LoadConfig() *Config {
	return &Config{
		DatabaseURL: os.Getenv("DATABASE_URL"), // Use env variable for DB connection URL
	}
}

func SetupDatabase(cfg *Config) *gorm.DB {
	db, err := gorm.Open("sqlite3", cfg.DatabaseURL)
	if err != nil {
		log.Fatalf("Error connecting to the database: %v", err)
	}
	return db
}
```

### 5. Domain Layer (domain/)
The domain layer will contain all your models and entities. Here, for instance, is how the User model might look.

`domain/user/model.go`

```go
package user

type User struct {
	ID    uint   `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}
```


### 6. Services Layer (app/)
Services implement business logic. They use repositories to interact with the database.

`app/user/service.go`

```go
package user

import (
	"go-clean-architecture/domain/user"
	"github.com/jinzhu/gorm"
)

type Service interface {
	GetAllUsers() ([]user.User, error)
}

type service struct {
	db *gorm.DB
}

func NewService(db *gorm.DB) Service {
	return &service{db: db}
}

func (s *service) GetAllUsers() ([]user.User, error) {
	var users []user.User
	err := s.db.Find(&users).Error
	return users, err
}
```

User Repository Example :

`app/user/repository.go`

```go
package user

import "go-clean-architecture/domain/user"

type Repository interface {
	GetUsers() ([]user.User, error)
}

type repository struct {
	// Database connection
}

func (r *repository) GetUsers() ([]user.User, error) {
	// Database querying logic
}
```

### 7. Error Handling
In your `domain/common/error.go`, define a custom error handler:

```go
package common

import "fmt"

type AppError struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
}

func (e *AppError) Error() string {
	return fmt.Sprintf("Error code: %d, message: %s", e.Code, e.Message)
}

func NewAppError(code int, message string) *AppError {
	return &AppError{Code: code, Message: message}
}
```

### 8. Logging & Monitoring
In `domain/common/logger.go`:

```go
package common

import (
	"log"
)

func LogInfo(message string) {
	log.Println("INFO: " + message)
}

func LogError(err error) {
	log.Println("ERROR: " + err.Error())
}
```

> **ProTip:** For monitoring, you can integrate with something like Prometheus. For logging, you can use structured logging libraries like logrus.

### 9. Migrations & Seeds (migrations/)

**Migrations**

```go
package migrations

import (
	"github.com/jinzhu/gorm"
	"go-clean-architecture/domain/user"
)

func RunMigrations(db *gorm.DB) {
	// Apply migration here
	db.AutoMigrate(&user.User{})
}
```

**Seeding**

```go
package seed

import (
	"github.com/jinzhu/gorm"
	"go-clean-architecture/domain/user"
)

func SeedUsers(db *gorm.DB) {
	users := []user.User{
		{Name: "Alice", Email: "alice@example.com"},
		{Name: "Bob", Email: "bob@example.com"},
	}
	for _, user := range users {
		db.Create(&user)
	}
}
```


### 10.  Fiber Routes & Handlers

You can define routes for each service using Go Fiber's routing system.


`app/user/handler.go`

```go
package user

import (
	"github.com/gofiber/fiber/v2"
	"go-clean-architecture/domain/user"
)

func RegisterRoutes(app *fiber.App, service Service) {
	app.Get("/users", func(c *fiber.Ctx) error {
		users, err := service.GetAllUsers()
		if err != nil {
			return c.Status(500).JSON(user.AppError{Message: err.Error()})
		}
		return c.JSON(users)
	})
}
```

### 11.  System Tests + Mocking

For testing, we’ll use testify for assertions and mocks.


`test/user_service_test.go`

```go
package user

import (
	"testing"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

type MockDatabase struct {
	mock.Mock
}

func (m *MockDatabase) Find(dest interface{}, conds ...interface{}) error {
	args := m.Called(dest)
	return args.Error(0)
}

func TestGetAllUsers(t *testing.T) {
	mockDb := new(MockDatabase)
	mockDb.On("Find", mock.Anything).Return(nil)

	service := NewService(mockDb)
	users, err := service.GetAllUsers()

	assert.NoError(t, err)
	assert.Len(t, users, 0)
	mockDb.AssertExpectations(t)
}
```

### 12.  Running the Application

After setting up all these components, you can run your Go application by:

1. Initializing your database (via migration).

2. Seeding data.

3. Running the server :

```bash
go run app/main.go
```

## Conclusion

This setup demonstrates how to organize the project using Clean Architecture, SOLID principles, and the requested features such as dependency injection, GORM integration, migrations, and system testing. You can expand each service to handle more complex operations, error handling, logging, and monitoring as needed.