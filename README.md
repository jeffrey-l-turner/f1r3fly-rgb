# F1r3fly-RGB

## Setup

Clone the project:
```bash
git clone <repository-url>
cd f1r3fly-rgb
```

Initialize submodules:
```bash
git submodule update --init --recursive
```

Update submodules:
```bash
git submodule update --remote
```

## Running on Localhost

This project consists of two components that need to be running:

### 1. Backend (Rust)

The backend is a Rust HTTP server that provides the wallet API.

```bash
cd wallet
cargo run --release
```

The backend server will start at **http://127.0.0.1:3000**

You should see: `Starting RGB-compatible Bitcoin wallet server on 127.0.0.1:3000`

### 2. Frontend (React)

The frontend is a React + TypeScript application.

**First time setup:**
```bash
cd wallet-frontend
npm install
```

**Start the development server:**
```bash
cd wallet-frontend
npm run dev
```

The frontend will start at **http://localhost:5173**

Open your browser to http://localhost:5173 to use the wallet interface.

### Development Workflow

1. Start the backend in one terminal:
   ```bash
   cd wallet && cargo run --release
   ```

2. Start the frontend in another terminal:
   ```bash
   cd wallet-frontend && npm run dev
   ```

3. The frontend will automatically connect to the backend API at http://localhost:3000

### Building for Production

**Backend:**
```bash
cd wallet
cargo build --release
./target/release/wallet
```

**Frontend:**
```bash
cd wallet-frontend
npm run build
# Output will be in dist/ folder
```

## Testing

Run backend tests:
```bash
cd wallet
cargo test --release
```

Run frontend linter:
```bash
cd wallet-frontend
npm run lint
```

Run all tests including submodules (excluding f1r3node):
```bash
cargo test --release && git submodule foreach --recursive 'if [ "$name" != "f1r3node" ] && [ -f Cargo.toml ]; then echo "Testing $name"; cargo test --release; fi'
```
