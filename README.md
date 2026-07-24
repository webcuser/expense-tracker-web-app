# Expense Tracker Web App

This is a monorepo containing the backend and frontend for the Expense Tracker application.

## Setup

1. **Clone the repository**

```bash
git clone <repository-url>
cd <repository-directory>
```

2. **Start the Docker containers**

```bash
docker-compose up --build
```

3. **Access the application**
   - Laravel API: [http://localhost/api](http://localhost/api)
   - Next.js Frontend: [http://localhost:3000](http://localhost:3000)
   - Mailpit: [http://localhost:8025](http://localhost:8025)

## Development

- Backend is a Laravel application located in the `backend` directory.
- Frontend is a Next.js application located in the `frontend` directory.

## Running Migrations

To run migrations, access the Laravel container and run:

```bash
docker-compose exec php php artisan migrate
```
