# webapp-rpc-framework

A high-performance, type-safe RPC (Remote Procedure Call) framework for building enterprise-grade distributed systems in Rust. This library provides a comprehensive toolkit for implementing JSON-RPC services with strong type guarantees, sophisticated error handling, and standardized response formats.

[![Crates.io](https://img.shields.io/crates/v/webapp-rpc-framework.svg)](https://crates.io/crates/webapp-rpc-framework)
[![Documentation](https://docs.rs/webapp-rpc-framework/badge.svg)](https://docs.rs/webapp-rpc-framework)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## Overview

`webapp-rpc-framework` is designed to streamline the development of RPC services by providing:

- Type-safe parameter handling with compile-time guarantees
- Automated CRUD operation generation through powerful macros
- Standardized error handling with sophisticated error propagation
- Flexible filtering and pagination for list operations
- Comprehensive integration with async Rust ecosystem

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Architecture](#architecture)
- [Usage Guide](#usage-guide)
  - [Basic Setup](#basic-setup)
  - [Type-Safe Parameters](#type-safe-parameters)
  - [Error Handling](#error-handling)
  - [Macro System](#macro-system)
  - [Advanced Features](#advanced-features)
- [Examples](#examples)
- [API Documentation](#api-documentation)
- [Contributing](#contributing)
- [License](#license)

## Features

### Type-Safe RPC Infrastructure

- **Parameter Validation**: Compile-time parameter validation through generic type constraints
- **Response Standardization**: Consistent response formats across all RPC endpoints
- **Error Propagation**: Sophisticated error handling with automatic conversion and propagation
- **Async Support**: Built on `tokio` for high-performance async operations

### Advanced Querying Capabilities

- **Flexible Filtering**: Powerful query building with `modql` integration
- **Pagination Support**: Built-in pagination with cursor and offset-based options
- **Sort Operations**: Flexible sorting with multiple field support
- **Query Optimization**: Efficient query execution through smart caching and query planning

### Macro System

The library provides a sophisticated macro system for reducing boilerplate while maintaining type safety and consistency:

```rust
generate_common_rpc_fns! {
    Bmc: UserBmc,
    Entity: User,
    ForCreate: CreateUser,
    ForUpdate: UpdateUser,
    Filter: UserFilter,
    Suffix: user
}
```

Generated endpoints automatically handle:
- Parameter validation and deserialization
- Error handling and propagation
- Response serialization and formatting
- Database operations through the Business Model Controller (BMC)

## Installation

Add the following to your `Cargo.toml`:

```toml
[dependencies]
webapp-rpc-framework = "0.1.0"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

## Architecture

### Core Components

```plaintext
webapp-rpc-framework/
├── error/
│   └── Centralized error handling and propagation
├── rpc_params/
│   └── Type-safe parameter definitions
├── rpc_result/
│   └── Standardized response formatting
├── macro_utils/
│   └── Code generation utilities
└── prelude/
    └── Convenient imports and re-exports
```

### Type System

The library employs a sophisticated type system ensuring compile-time correctness:

```rust
// Parameter Types
pub struct ParamsForCreate<D> {
    pub data: D,
}

pub struct ParamsForUpdate<D> {
    pub id: i64,
    pub data: D,
}

pub struct ParamsIded {
    pub id: i64,
}

pub struct ParamsList<F> {
    pub filters: Option<Vec<F>>,
    pub list_options: Option<ListOptions>,
}

// Response Types
pub struct DataRpcResult<T> {
    data: T,
}
```

### Error Handling

Comprehensive error handling system with automatic conversions:

```rust
#[derive(Debug, From, Serialize, RpcHandlerError)]
pub enum Error {
    #[from]
    Model(lib_core::model::Error),

    #[from]
    SerdeJson(serde_json::Error),
}
```

## Usage Guide

### Basic Setup

1. Import the prelude:
```rust
use webapp_rpc_core::prelude::*;
```

2. Define your entity and associated types:
```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct User {
    id: i64,
    email: String,
    status: UserStatus,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateUser {
    email: String,
    status: UserStatus,
}

#[derive(Debug, Deserialize)]
pub struct UpdateUser {
    email: Option<String>,
    status: Option<UserStatus>,
}

#[derive(Debug, Deserialize, Default)]
pub struct UserFilter {
    status: Option<UserStatus>,
    email_contains: Option<String>,
}
```

3. Generate RPC handlers:
```rust
generate_common_rpc_fns! {
    Bmc: UserBmc,
    Entity: User,
    ForCreate: CreateUser,
    ForUpdate: UpdateUser,
    Filter: UserFilter,
    Suffix: user
}
```

### Generated Endpoints

The macro generates the following endpoints:

```rust
// Create a new user
async fn create_user(
    ctx: Ctx,
    mm: ModelManager,
    params: ParamsForCreate<CreateUser>,
) -> Result<DataRpcResult<User>>

// Retrieve a user by ID
async fn get_user(
    ctx: Ctx,
    mm: ModelManager,
    params: ParamsIded,
) -> Result<DataRpcResult<User>>

// List users with filtering and pagination
async fn list_users(
    ctx: Ctx,
    mm: ModelManager,
    params: ParamsList<UserFilter>,
) -> Result<DataRpcResult<Vec<User>>>

// Update an existing user
async fn update_user(
    ctx: Ctx,
    mm: ModelManager,
    params: ParamsForUpdate<UpdateUser>,
) -> Result<DataRpcResult<User>>

// Delete a user
async fn delete_user(
    ctx: Ctx,
    mm: ModelManager,
    params: ParamsIded,
) -> Result<DataRpcResult<User>>
```

### Advanced Features

#### Custom Filter Implementation

```rust
#[derive(Debug, Deserialize)]
pub struct ComplexUserFilter {
    #[serde(flatten)]
    base: UserFilter,
    age_range: Option<Range<i32>>,
    locations: Option<Vec<String>>,
}

impl Into<QueryFilter> for ComplexUserFilter {
    fn into(self) -> QueryFilter {
        let mut filter = QueryFilter::default();
        // Add sophisticated filtering logic
        filter
    }
}
```

#### Transaction Support

```rust
pub async fn create_user_with_preferences(
    ctx: &Ctx,
    mm: &ModelManager,
    params: ParamsForCreate<CreateUserWithPreferences>,
) -> Result<DataRpcResult<User>> {
    mm.transaction(|tx| async {
        let user = UserBmc::create(ctx, tx, params.data.user).await?;
        let prefs = PreferencesBmc::create(ctx, tx, params.data.preferences).await?;
        Ok(DataRpcResult::from(user))
    }).await
}
```

## Performance Considerations

- **Async by Default**: Built on `tokio` for non-blocking I/O
- **Connection Pooling**: Efficient database connection management
- **Query Optimization**: Smart query planning and execution
- **Caching Integration**: Ready for integration with caching layers

## Security

- **Context-Aware**: All operations require a valid context
- **Type-Safe Parameters**: Prevents injection attacks through strong typing
- **Error Sanitization**: Sensitive information is stripped from error messages
- **Rate Limiting Ready**: Designed for integration with rate limiting middleware

## Testing

The library includes comprehensive testing utilities:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use webapp_rpc_core::test_utils::*;

    #[tokio::test]
    async fn test_create_user() {
        let ctx = test_context();
        let mm = test_model_manager().await;
        
        let params = ParamsForCreate {
            data: CreateUser {
                email: "test@example.com".to_string(),
                status: UserStatus::Active,
            },
        };

        let result = create_user(ctx, mm, params).await.unwrap();
        assert_eq!(result.data.email, "test@example.com");
    }
}
```

## API Documentation

Comprehensive API documentation is available at [docs.rs](https://docs.rs/webapp-rpc-framework).

## Contributing

This is currently a personal project developed and maintained by eohyungk. If you find any issues or have suggestions, please feel free to open an issue on the repository.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Built with [tokio](https://tokio.rs/) for async runtime
- Uses [serde](https://serde.rs/) for serialization
- Integrates with [modql](https://crates.io/crates/modql) for query building

---

Built with ❤️ by eohyungk
