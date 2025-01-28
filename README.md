# Setting Up PostgreSQL with Go for a CRUD Application

## Step 1: Set Up PostgreSQL

### Install PostgreSQL
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

### Start the PostgreSQL Service
```bash
sudo systemctl start postgresql
```

### Enable PostgreSQL to Start at Boot
```bash
sudo systemctl enable postgresql
```

### Switch to the PostgreSQL User and Access the PostgreSQL Shell
```bash
sudo -i -u postgres
psql
```

### Create a Database
```sql
CREATE DATABASE crud_db;
```

### Create a `products` Table
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price NUMERIC(10,2) NOT NULL,
    quantity INT NOT NULL
);
```

### Exit the PostgreSQL Shell
```bash
\q
```

## Step 2: Install Dependencies

### Initialize Your Go Project and Install Required Modules
```bash
go mod init github.com/mjpvl-ai/golangdb
go get -u github.com/gorilla/mux
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
```

## Step 3: Create the Main Go File

### Create a File Named `main.go`
```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"

	"github.com/gorilla/mux"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
)

// Product represents the product model
type Product struct {
	ID       uint    `json:"id" gorm:"primaryKey"`
	Name     string  `json:"name"`
	Price    float64 `json:"price"`
	Quantity int     `json:"quantity"`
}

var db *gorm.DB

func initDB() {
	var err error
	// Set up PostgreSQL connection
	dsn := "host=localhost user=postgres password=yourpassword dbname=crud_db port=5432 sslmode=disable"
	db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatal("Failed to connect to database:", err)
	}

	// Migrate the Product model
	err = db.AutoMigrate(&Product{})
	if err != nil {
		log.Fatal("Failed to migrate database:", err)
	}
	fmt.Println("Database connected and migrated!")
}

// Handlers for CRUD Operations

// Get all products
func getProducts(w http.ResponseWriter, r *http.Request) {
	var products []Product
	db.Find(&products)
	json.NewEncoder(w).Encode(products)
}

// Get a single product by ID
func getProduct(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)
	var product Product
	if err := db.First(&product, params["id"]).Error; err != nil {
		w.WriteHeader(http.StatusNotFound)
		json.NewEncoder(w).Encode(map[string]string{"error": "Product not found"})
		return
	}
	json.NewEncoder(w).Encode(product)
}

// Create a new product
func createProduct(w http.ResponseWriter, r *http.Request) {
	var product Product
	if err := json.NewDecoder(r.Body).Decode(&product); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(map[string]string{"error": "Invalid request payload"})
		return
	}
	db.Create(&product)
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(product)
}

// Update an existing product
func updateProduct(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)
	var product Product
	if err := db.First(&product, params["id"]).Error; err != nil {
		w.WriteHeader(http.StatusNotFound)
		json.NewEncoder(w).Encode(map[string]string{"error": "Product not found"})
		return
	}
	var updatedProduct Product
	if err := json.NewDecoder(r.Body).Decode(&updatedProduct); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(map[string]string{"error": "Invalid request payload"})
		return
	}
	product.Name = updatedProduct.Name
	product.Price = updatedProduct.Price
	product.Quantity = updatedProduct.Quantity
	db.Save(&product)
	json.NewEncoder(w).Encode(product)
}

// Delete a product by ID
func deleteProduct(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)
	var product Product
	if err := db.First(&product, params["id"]).Error; err != nil {
		w.WriteHeader(http.StatusNotFound)
		json.NewEncoder(w).Encode(map[string]string{"error": "Product not found"})
		return
	}
	db.Delete(&product)
	w.WriteHeader(http.StatusNoContent)
}

// Main function
func main() {
	initDB()

	router := mux.NewRouter()
	router.HandleFunc("/products", getProducts).Methods("GET")
	router.HandleFunc("/products/{id}", getProduct).Methods("GET")
	router.HandleFunc("/products", createProduct).Methods("POST")
	router.HandleFunc("/products/{id}", updateProduct).Methods("PUT")
	router.HandleFunc("/products/{id}", deleteProduct).Methods("DELETE")

	fmt.Println("Server running on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", router))
}
```

## Step 4: Run the Application

Replace `yourpassword` in the `dsn` variable with your PostgreSQL password, then run the application:
```bash
go run main.go
```

The server will start at [http://localhost:8080](http://localhost:8080).

## Step 5: Test the API

You can test the API using tools like Postman or `curl`.

### Create a Product
```bash
curl -X POST -H "Content-Type: application/json" \
	-d '{"name": "Laptop", "price": 1500.50, "quantity": 10}' \
	http://localhost:8080/products
```

### Get All Products
```bash
curl http://localhost:8080/products
```

### Get a Product by ID
```bash
curl http://localhost:8080/products/1
```

### Update a Product
```bash
curl -X PUT -H "Content-Type: application/json" \
	-d '{"name": "Gaming Laptop", "price": 2000.00, "quantity": 5}' \
	http://localhost:8080/products/1
```

### Delete a Product
```bash
curl -X DELETE http://localhost:8080/products/1
