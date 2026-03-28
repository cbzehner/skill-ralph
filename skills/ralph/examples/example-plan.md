---
status: pending
gaps: []
edge_cases: []
progress: []
last_review: null
---

# Example Feature: User Authentication

This is an example plan file showing the expected format for `/ralph`.

## Section 1: Database Schema

Create the users table with the following columns:
- `id` (UUID, primary key)
- `email` (string, unique, not null)
- `password_hash` (string, not null)
- `created_at` (timestamp)
- `updated_at` (timestamp)

Write the migration file and run it.

## Section 2: User Model

Create a User model/struct with:
- Validation for email format
- Password hashing on create
- Method to verify password

## Section 3: Auth Endpoints

Implement REST endpoints:
- `POST /auth/register` - Create new user
- `POST /auth/login` - Authenticate and return JWT
- `POST /auth/logout` - Invalidate session

## Section 4: Middleware

Create auth middleware that:
- Extracts JWT from Authorization header
- Validates the token
- Attaches user to request context
- Returns 401 for invalid/missing tokens

## Section 5: Tests

Write tests for:
- User model validation
- Password hashing/verification
- Each auth endpoint
- Middleware behavior
