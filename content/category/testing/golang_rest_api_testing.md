Title: Building & Testing a REST API in Go using httpserver, httptest, testing & testify
Category: Testing
Date: 2025-06-02
Tags: go, go testing, rest, testing, http, testify, httpserver, httptest
Slug: go-server-rest-api-test
Summary: A quick walkthrough of building and testing a tiny REST API in Go using the standard library and testify.

In this post, we’ll build a minimal (just for testing purposes) REST API in Go with in-memory storage using `httpserver` and test it using Go’s standard `testing` package, `httptest` as a part of `httpserver` and the `testify` library.

The complete source code is available on GitHub:  
[https://github.com/emkeyen/go_server_test_api](https://github.com/emkeyen/go_server_test_api){:target="_blank"}

Just clone the repo and run: `go mod tidy`

This installs all dependencies listed in the `go.mod` and `go.sum` files.

## HTTP Server Code

This simple Go server keeps all user data in memory using a map protected by a mutex to avoid race conditions. It’s fast and lightweight since there’s no database - everything lives in RAM and resets when the server restarts.

For testing, it starts with one user (ID 1, "Test User1"). You get three main endpoints:

- `/` - just a quick welcome message.
- `/hello` - says hello, only accepts GET.
- `/user` - handles user creation, reading, updating, and deleting via JSON and query params.

The server carefully checks HTTP methods and input data, returning proper errors if something’s off. Using a mutex means multiple requests won’t mess up the user data at the same time.

This setup is perfect for quick testing or learning HTTP in Go without setting up a database.

Here’s the code:

    :::golang
    package httpserver

    import (
        "encoding/json"
        "net/http"
        "strconv"
        "sync"
    )

    var (
        Users  = make(map[int]User)
        Mu     sync.RWMutex
        NextID = 1
    )

    type User struct {
        ID   int    `json:"id"`
        Name string `json:"name"`
    }

    func GetRoot(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/" {
            http.NotFound(w, r)
            return
        }
        w.Write([]byte("This is a simple Go http server :)\n"))
    }

    func GetHello(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodGet {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }
        w.Write([]byte("Hello, HTTP!\n"))
    }

    func UserHandler(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodGet:
            handleGetUser(w, r)
        case http.MethodPost:
            handleCreateUser(w, r)
        case http.MethodPatch:
            handleUpdateUser(w, r)
        case http.MethodDelete:
            handleDeleteUser(w, r)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
    }

    func handleGetUser(w http.ResponseWriter, r *http.Request) {
        idStr := r.URL.Query().Get("id")
        if idStr == "" {
            http.Error(w, "Missing user ID", http.StatusBadRequest)
            return
        }

        id, err := strconv.Atoi(idStr)
        if err != nil {
            http.Error(w, "Invalid user ID", http.StatusBadRequest)
            return
        }

        Mu.RLock()
        user, exists := Users[id]
        Mu.RUnlock()

        if !exists {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(user)
    }

    func handleCreateUser(w http.ResponseWriter, r *http.Request) {
        var user User
        if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
            http.Error(w, "Invalid user data", http.StatusBadRequest)
            return
        }

        if user.Name == "" {
            http.Error(w, "Name is required", http.StatusBadRequest)
            return
        }

        Mu.Lock()
        defer Mu.Unlock()

        if user.ID == 0 {
            user.ID = NextID
            NextID++
        }

        if _, exists := Users[user.ID]; exists {
            http.Error(w, "User already exists", http.StatusConflict)
            return
        }

        Users[user.ID] = user
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(user)
    }

    func handleUpdateUser(w http.ResponseWriter, r *http.Request) {
        var user User
        if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
            http.Error(w, "Invalid user data", http.StatusBadRequest)
            return
        }

        Mu.Lock()
        defer Mu.Unlock()

        _, exists := Users[user.ID]
        if !exists {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }

        Users[user.ID] = user
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(user)
    }

    func handleDeleteUser(w http.ResponseWriter, r *http.Request) {
        idStr := r.URL.Query().Get("id")
        if idStr == "" {
            http.Error(w, "Missing user ID", http.StatusBadRequest)
            return
        }

        id, err := strconv.Atoi(idStr)
        if err != nil {
            http.Error(w, "Invalid user ID", http.StatusBadRequest)
            return
        }

        Mu.Lock()
        defer Mu.Unlock()

        if _, exists := Users[id]; !exists {
            http.Error(w, "User not found :<", http.StatusNotFound)
            return
        }

        delete(Users, id)
        w.WriteHeader(http.StatusNoContent)
    }


## Test File

The test file below uses Go’s built-in `testing` package to organize and run tests, and the `httptest` package to fake HTTP requests without starting a real server, so tests run fast and isolated.
The `testify` library helps write clear and expressive assertions, making tests easier to read and maintain.

The tests cover all main user actions (create, read, update, delete) plus edge cases like missing or invalid data. The `testing` package runs the test functions, while `httptest` mocks requests and responses. Importantly, the server never actually runs during tests, so they’re quick and safe.

This setup ensures your API behaves as expected before you deploy or use it for real.

These tests cover all key endpoints and edge cases - creating, reading, updating, and deleting users - making sure your API works smoothly.

Just run `go test ./httpserver -v` and see it in action.

    :::golang
    package httpserver

    import (
        "bytes"
        "encoding/json"
        "net/http"
        "net/http/httptest"
        "os"
        "strconv"
        "testing"

        "github.com/stretchr/testify/assert"
        "github.com/stretchr/testify/require"
    )

    func TestMain(m *testing.M) {
        Mu.Lock()
        Users = make(map[int]User)
        Users[1] = User{ID: 1, Name: "Test User1"}
        NextID = 2
        Mu.Unlock()
        os.Exit(m.Run())
    }

    func TestRootEndpoint(t *testing.T) {
        req := httptest.NewRequest("GET", "/", nil)
        rr := httptest.NewRecorder()
        GetRoot(rr, req)

        assert.Equal(t, http.StatusOK, rr.Code)
        assert.Contains(t, rr.Body.String(), "This is a simple Go http server")
    }

    func TestHelloEndpoint(t *testing.T) {
        req := httptest.NewRequest("GET", "/hello", nil)
        rr := httptest.NewRecorder()
        GetHello(rr, req)

        assert.Equal(t, http.StatusOK, rr.Code)
        assert.Contains(t, rr.Body.String(), "Hello, HTTP!")
    }

    func TestCreateUser(t *testing.T) {
        newUser := User{Name: "Integration Test User"}
        jsonData, err := json.Marshal(newUser)
        require.NoError(t, err)

        req := httptest.NewRequest("POST", "/user", bytes.NewBuffer(jsonData))
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusCreated, rr.Code)

        var createdUser User
        err = json.Unmarshal(rr.Body.Bytes(), &createdUser)
        require.NoError(t, err)

        assert.NotZero(t, createdUser.ID)
        assert.Equal(t, newUser.Name, createdUser.Name)

        Mu.RLock()
        defer Mu.RUnlock()
        storedUser, exists := Users[createdUser.ID]
        assert.True(t, exists)
        assert.Equal(t, createdUser, storedUser)
    }

    func TestGetUser(t *testing.T) {
        Mu.Lock()
        testUser := User{ID: 100, Name: "Test Get User"}
        Users[testUser.ID] = testUser
        Mu.Unlock()

        req := httptest.NewRequest("GET", "/user?id="+strconv.Itoa(testUser.ID), nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusOK, rr.Code)

        var retrievedUser User
        err := json.Unmarshal(rr.Body.Bytes(), &retrievedUser)
        require.NoError(t, err)

        assert.Equal(t, testUser, retrievedUser)
    }

    func TestGetUserNotFound(t *testing.T) {
        req := httptest.NewRequest("GET", "/user?id=9999", nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusNotFound, rr.Code)
    }

    func TestUpdateUser(t *testing.T) {
        Mu.Lock()
        testUser := User{ID: 200, Name: "Before Update"}
        Users[testUser.ID] = testUser
        Mu.Unlock()

        updatedUser := User{ID: 200, Name: "After Update"}
        jsonData, err := json.Marshal(updatedUser)
        require.NoError(t, err)

        req := httptest.NewRequest("PATCH", "/user", bytes.NewBuffer(jsonData))
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusOK, rr.Code)

        var responseUser User
        err = json.Unmarshal(rr.Body.Bytes(), &responseUser)
        require.NoError(t, err)

        assert.Equal(t, updatedUser, responseUser)

        Mu.RLock()
        defer Mu.RUnlock()
        storedUser, exists := Users[200]
        assert.True(t, exists)
        assert.Equal(t, updatedUser, storedUser)
    }

    func TestUpdateUserNotFound(t *testing.T) {
        nonExistentUser := User{ID: 9999, Name: "Non-existent"}
        jsonData, err := json.Marshal(nonExistentUser)
        require.NoError(t, err)

        req := httptest.NewRequest("PATCH", "/user", bytes.NewBuffer(jsonData))
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusNotFound, rr.Code)
    }

    func TestDeleteUser(t *testing.T) {
        Mu.Lock()
        testUser := User{ID: 300, Name: "To Be Deleted"}
        Users[testUser.ID] = testUser
        Mu.Unlock()

        req := httptest.NewRequest("DELETE", "/user?id="+strconv.Itoa(testUser.ID), nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusNoContent, rr.Code)

        Mu.RLock()
        defer Mu.RUnlock()
        _, exists := Users[300]
        assert.False(t, exists)
    }

    func TestDeleteUserNotFound(t *testing.T) {
        req := httptest.NewRequest("DELETE", "/user?id=9999", nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusNotFound, rr.Code)
        assert.Contains(t, rr.Body.String(), "User not found :<")
    }

    func TestCreateUserWithExistingID(t *testing.T) {
        Mu.Lock()
        testUser := User{ID: 400, Name: "Existing User"}
        Users[testUser.ID] = testUser
        Mu.Unlock()

        duplicateUser := User{ID: 400, Name: "Duplicate"}
        jsonData, err := json.Marshal(duplicateUser)
        require.NoError(t, err)

        req := httptest.NewRequest("POST", "/user", bytes.NewBuffer(jsonData))
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusConflict, rr.Code)
    }

    func TestCreateUserInvalidData(t *testing.T) {
        invalidUser := User{Name: ""}
        jsonData, err := json.Marshal(invalidUser)
        require.NoError(t, err)

        req := httptest.NewRequest("POST", "/user", bytes.NewBuffer(jsonData))
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusBadRequest, rr.Code)
    }

    func TestMethodNotAllowed(t *testing.T) {
        req := httptest.NewRequest("PUT", "/user", nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusMethodNotAllowed, rr.Code)
    }

    func TestUserHandlerInvalidPath(t *testing.T) {
        req := httptest.NewRequest("GET", "/invalid", nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusBadRequest, rr.Code)
    }

    func TestGetUserMissingID(t *testing.T) {
        req := httptest.NewRequest("GET", "/user", nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusBadRequest, rr.Code)
        assert.Contains(t, rr.Body.String(), "Missing user ID")
    }

    func TestDeleteUserMissingID(t *testing.T) {
        req := httptest.NewRequest("DELETE", "/user", nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusBadRequest, rr.Code)
        assert.Contains(t, rr.Body.String(), "Missing user ID")
    }

    func TestGetUserInvalidID(t *testing.T) {
        req := httptest.NewRequest("GET", "/user?id=invalid", nil)
        rr := httptest.NewRecorder()
        UserHandler(rr, req)

        assert.Equal(t, http.StatusBadRequest, rr.Code)
        assert.Contains(t, rr.Body.String(), "Invalid user ID")
    }


## Main 
To wire everything up, we just need a `main.go` that sets up the initial user data and registers the HTTP handlers. 
The server listens on port :3333 and logs when it starts.
Here’s what `main.go` looks like:

    :::golang
    package main

    import (
        "log"
        "net/http"

        "github.com/emkeyen/go_server_test_api/httpserver"
    )

    func main() {
        // init with test data
        httpserver.Mu.Lock()
        httpserver.Users[1] = httpserver.User{ID: 1, Name: "Test User1"}
        httpserver.NextID = 2
        httpserver.Mu.Unlock()

        // register handlers
        http.HandleFunc("/", httpserver.GetRoot)
        http.HandleFunc("/hello", httpserver.GetHello)
        http.HandleFunc("/user", httpserver.UserHandler)

        log.Println("Starting server on :3333")
        log.Fatal(http.ListenAndServe(":3333", nil))
    }

#### Run Tests

    :::bash
    go test ./httpserver -v

#### Start the Server

    :::bash
    go run main.go
    # Server running on http://localhost:3333

### API Endpoints

#### User CRUD Operations

**Create User (POST)**

    :::bash
    curl -X POST http://localhost:3333/user \
      -H "Content-Type: application/json" \
      -d '{"name":"New User"}'

**Get User (GET)**

    :::bash
    curl "http://localhost:3333/user?id=1"

**Update User (PATCH)**

    :::bash
    curl -X PATCH http://localhost:3333/user \
      -H "Content-Type: application/json" \
      -d '{"id":1,"name":"Updated Name"}'

**Delete User (DELETE)**

    :::bash
    curl -X DELETE "http://localhost:3333/user?id=1"

#### Utility Endpoints

**Root Endpoint**

    :::bash
    curl http://localhost:3333/

**Hello Endpoint**

    :::bash
    curl http://localhost:3333/hello
