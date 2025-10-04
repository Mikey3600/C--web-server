# C--web-server
A simple HTTP web server implemented in C for Windows using the Winsock2 API. This server can handle basic GET requests and serve static files (HTML, CSS, JS, images, etc.) from the local directory. It demonstrates socket programming, HTTP response handling, MIME type detection, and file I/O in c.
# C HTTP Web Server using Winsock

A simple HTTP web server implemented in C for Windows using the Winsock2 API. This server handles basic GET requests and serves static files (HTML, CSS, JS, images, etc.) from the local directory. It demonstrates socket programming, HTTP response handling, MIME type detection, and file I/O in C. Perfect for learning low-level network programming on Windows.

---

## Features

- Handles multiple client connections (basic queue).  
- Serves static files with correct MIME types.  
- Sends standard HTTP responses: `200 OK`, `404 Not Found`, `500 Internal Server Error`.  
- Built using **pure C** and **Winsock2**, no external libraries required.  
- Simple, easy-to-read code suitable for beginners learning network programming.

---

## Prerequisites

- Windows OS  
- Visual Studio or GCC/MinGW compiler  

---

## How to Compile & Run

### Using GCC (MinGW)
```bash
gcc server.c -o server.exe -lws2_32
./server.exe
