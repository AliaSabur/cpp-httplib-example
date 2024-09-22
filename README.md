# cpp-httplib-example

# Project Description

This project uses **`cpp-httplib`** and **`nlohmann::json`** libraries to implement both an HTTP server and client within the same program. The server and client run concurrently using multithreading. The client sends various HTTP requests to test the server's functionality.

## Dependencies

- **`cpp-httplib`**: A single-header C++ HTTP/HTTPS server and client library.

  - GitHub repository: [https://github.com/yhirose/cpp-httplib](https://github.com/yhirose/cpp-httplib)
  - Download the latest `httplib.h` file and include it in your project directory.

- **`nlohmann::json`**: A modern C++ JSON library.

  - GitHub repository: [https://github.com/nlohmann/json](https://github.com/nlohmann/json)
  - Download the latest `json.hpp` file and include it in your project directory.

## Compilation

Please use a compiler that supports the **C++20** standard.

# Example usage

```c++
// main.cpp

#include <iostream>
#include <thread>
#include <chrono>
#include "json.hpp"
#include "httplib.h"

// GET request handler
void handle_get_data(const httplib::Request& req, httplib::Response& res) {
	std::string folder = req.get_param_value("folder");
	nlohmann::json json_obj = {
		{"message", "Get Data"},
		{"folder", folder},
		{"path", req.path},
		{"query_params", req.params}
	};
	res.set_content(json_obj.dump(), "application/json");
}

// POST request handler
void handle_post_data(const httplib::Request& req, httplib::Response& res) {
	// Parse JSON request body
	try {
		auto json_body = nlohmann::json::parse(req.body);
		// Print client's JSON content
		std::cout << "Received JSON data from client: " << json_body.dump() << std::endl;

		// Add server's fields
		nlohmann::json response_json = {
			{"message", "Data Received"},
			{"server", "MyServerName"},
			{"data", json_body}
		};
		res.status = 201; // Created
		res.set_content(response_json.dump(), "application/json");
	}
	catch (const std::exception& e) {
		nlohmann::json error_json = {
			{"error", "Invalid JSON data"},
			{"details", e.what()}
		};
		res.status = 400; // Bad Request
		res.set_content(error_json.dump(), "application/json");
	}
}

// PUT request handler
void handle_put_data(const httplib::Request& req, httplib::Response& res) {
	std::string data_id = req.matches[1]; // Extract path parameter
	try {
		auto json_body = nlohmann::json::parse(req.body);
		nlohmann::json response_json = {
			{"message", "Data Updated"},
			{"data_id", data_id},
			{"updated_data", json_body}
		};
		res.set_content(response_json.dump(), "application/json");
	}
	catch (const std::exception& e) {
		nlohmann::json error_json = {
			{"error", "Invalid JSON data"},
			{"details", e.what()}
		};
		res.status = 400; // Bad Request
		res.set_content(error_json.dump(), "application/json");
	}
}

// DELETE request handler
void handle_delete_data(const httplib::Request& req, httplib::Response& res) {
	std::string data_id = req.matches[1]; // Extract path parameter
	nlohmann::json response_json = {
		{"message", "Data Deleted"},
		{"data_id", data_id}
	};
	res.set_content(response_json.dump(), "application/json");
}

// PATCH request handler
void handle_patch_data(const httplib::Request& req, httplib::Response& res) {
	std::string data_id = req.matches[1]; // Extract path parameter
	try {
		auto json_body = nlohmann::json::parse(req.body);
		nlohmann::json response_json = {
			{"message", "Data Partially Updated"},
			{"data_id", data_id},
			{"updated_data", json_body}
		};
		res.set_content(response_json.dump(), "application/json");
	}
	catch (const std::exception& e) {
		nlohmann::json error_json = {
			{"error", "Invalid JSON data"},
			{"details", e.what()}
		};
		res.status = 400; // Bad Request
		res.set_content(error_json.dump(), "application/json");
	}
}

// OPTIONS request handler
void handle_options_data(const httplib::Request& req, httplib::Response& res) {
	res.status = 204; // No Content
	res.set_header("Allow", "GET, POST, PUT, DELETE, PATCH, OPTIONS");

	// Add CORS headers
	res.set_header("Access-Control-Allow-Origin", "*");
	res.set_header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, PATCH, OPTIONS");
	res.set_header("Access-Control-Allow-Headers", "Content-Type, Authorization");
	res.set_header("Access-Control-Allow-Credentials", "true");
}

// CORS preflight handler
void handle_cors(const httplib::Request& req, httplib::Response& res) {
	res.set_header("Access-Control-Allow-Origin", "*");
	res.set_header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, PATCH, OPTIONS");
	res.set_header("Access-Control-Allow-Headers", "Content-Type, Authorization");
	res.set_header("Access-Control-Allow-Credentials", "true");
}

int main() {

	httplib::Server server;

	// Set up CORS preflight handler
	server.set_pre_routing_handler([](const httplib::Request& req, httplib::Response& res) {
		handle_cors(req, res);
		// Always return Unhandled to allow the request to continue to route handlers
		return httplib::Server::HandlerResponse::Unhandled;
		});

	// Register routes and handlers
	server.Get(R"(/path1/path2)", handle_get_data);
	server.Get(R"(/fc92e6fddf/path1/path2)", handle_get_data);

	server.Post(R"(/data)", handle_post_data);
	server.Put(R"(/data/(\d+))", handle_put_data); // \d+ matches one or more digits
	server.Delete(R"(/data/(\d+))", handle_delete_data);
	server.Patch(R"(/data/(\d+))", handle_patch_data);

	// Remove HEAD request handler
	// No need to register HEAD request handler; server handles it automatically

	server.Options(R"(/data)", handle_options_data);

	// Start the server
	// std::cout << "Server is running on http://localhost:28080/. Press Ctrl+C to stop." << std::endl;
	// server.listen("0.0.0.0", 28080);

	// Start server thread
	std::thread server_thread([&server]() {
		std::cout << "Server is running on http://localhost:28080/. Press Ctrl+C to stop." << std::endl;
		server.listen("0.0.0.0", 28080);
		});

	// Wait for the server to start
	std::this_thread::sleep_for(std::chrono::seconds(1));

	// Base URL
	std::string base_url = "http://localhost:28080";

	// Create client
	httplib::Client cli("localhost", 28080);

	// Use GET method with query parameters
	auto res_get = cli.Get("/path1/path2?folder=/app");
	if (res_get && res_get->status == 200) {
		auto json_res = nlohmann::json::parse(res_get->body);
		std::cout << "GET Response: " << json_res.dump(4) << std::endl;
	}
	else {
		std::cerr << "GET Request failed" << std::endl;
	}

	// Use GET method with complex path and query parameters
	auto res_get_complex = cli.Get("/fc92e6fddf/path1/path2?id=10086");
	if (res_get_complex && res_get_complex->status == 200) {
		auto json_res = nlohmann::json::parse(res_get_complex->body);
		std::cout << "GET Response with complex path: " << json_res.dump(4) << std::endl;
	}
	else {
		std::cerr << "GET Complex Request failed" << std::endl;
	}

	// Use POST method, send JSON data
	nlohmann::json json_post = { {"name", "Alice"}, {"age", 30} };
	auto res_post = cli.Post("/data", json_post.dump(), "application/json");
	if (res_post && res_post->status == 201) {
		auto json_res = nlohmann::json::parse(res_post->body);
		std::cout << "POST Response: " << json_res.dump(4) << std::endl;
	}
	else {
		std::cerr << "POST Request failed" << std::endl;
	}

	// Use PUT method to update data
	nlohmann::json json_put = { {"key", "updated_value"} };
	auto res_put = cli.Put("/data/1", json_put.dump(), "application/json");
	if (res_put && res_put->status == 200) {
		auto json_res = nlohmann::json::parse(res_put->body);
		std::cout << "PUT Response: " << json_res.dump(4) << std::endl;
	}
	else {
		std::cerr << "PUT Request failed" << std::endl;
	}

	// Use DELETE method to delete data
	auto res_delete = cli.Delete("/data/1");
	if (res_delete && res_delete->status == 200) {
		auto json_res = nlohmann::json::parse(res_delete->body);
		std::cout << "DELETE Response: " << json_res.dump(4) << std::endl;
	}
	else {
		std::cerr << "DELETE Request failed" << std::endl;
	}

	// Use OPTIONS method to get supported methods
	auto res_options = cli.Options("/data");
	if (res_options && res_options->status == 204) {
		std::string allow_methods = res_options->get_header_value("Allow");
		std::cout << "OPTIONS Allow Header: " << allow_methods << std::endl;
	}
	else {
		std::cerr << "OPTIONS Request failed" << std::endl;
	}

	// Use PATCH method to partially update data
	nlohmann::json json_patch = { {"key", "partial_update"} };
	auto res_patch = cli.Patch("/data/1", json_patch.dump(), "application/json");
	if (res_patch && res_patch->status == 200) {
		auto json_res = nlohmann::json::parse(res_patch->body);
		std::cout << "PATCH Response: " << json_res.dump(4) << std::endl;
	}
	else {
		std::cerr << "PATCH Request failed" << std::endl;
	}

	// Stop the server
	server.stop();
	if (server_thread.joinable()) {
		server_thread.join();
	}

	return 0;
}
```
