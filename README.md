## Getting Started

First, install the dependencies using pnpm:

```bash
pnpm install
```

Then, run the development server on port 3001:

```bash
pnpm dev --port 3001
```

Open [http://localhost:3001](http://localhost:3001) with your browser to see the result.

You can start editing the page by modifying `app/page.tsx`. The page auto-updates as you edit the file.

## Running the Exploit

This project contains a proof-of-concept exploit for a remote code execution vulnerability.

With the development server running on port 3001, open a new terminal and run the exploit script:

```bash
node rce-exploit.js
```

If successful, this will create a file named `test.txt` in the project's root directory with the content `hi how you doing`.
