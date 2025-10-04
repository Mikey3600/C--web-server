#include <stdio.h> //printf , perror , snprintf
#include <stdlib.h> // malloc , free , exit
#include <string.h> //strcmp, strcat, strlen,memset
#include <winsock2.h> //write , read , close
#include <ws2tcpip.h> // socket , bind , listen , accept
#include <io.h> // _close   ( close file descriptor )
#include <fcntl.h> // _O_BINARY ( open file in binary mode )
#include <sys/stat.h> // _fstat , struct _stat ( get file size )
#define MAX_PENDING 10  //Maximum number of pending connections in the queue

#pragma comment(lib, "ws2_32.lib") // Winsock Library

#define PORT 8080//The port on which to listen for incoming data
#define BUFFER_SIZE 1024//Buffer size for data transfer
#define QUEUE_SIZE 10  //Maximum number of pending connections in the queue

// HTTP response headers
const char *http_200 = "HTTP/1.1 200 OK\r\n";// Standard response for successful HTTP requests
const char *http_404 = "HTTP/1.1 404 Not Found\r\n";// The requested resource could not be found
const char *http_500 = "HTTP/1.1 500 Internal Server Error\r\n";// A generic error message, given when an unexpected condition was encountered

// Get MIME type based on file extension(Multipurpose Internet Mail Extensions)
const char *get_mime_type(const char *path) { // Function to determine the MIME type of a file based on its extension
    const char *ext = strrchr(path, '.');// Find the last occurrence of '.' in the file path
    if (!ext) return "application/octet-stream"; // Default MIME type

    if (strcmp(ext, ".html") == 0 || strcmp(ext, ".htm") == 0) return "text/html";
    if (strcmp(ext, ".css") == 0) return "text/css";
    if (strcmp(ext, ".js") == 0) return "application/javascript";
    if (strcmp(ext, ".png") == 0) return "image/png";
    if (strcmp(ext, ".jpg") == 0 || strcmp(ext, ".jpeg") == 0) return "image/jpeg";
    if (strcmp(ext, ".gif") == 0) return "image/gif";
    if (strcmp(ext, ".txt") == 0) return "text/plain";
    if (strcmp(ext, ".pdf") == 0) return "application/pdf";
    if (strcmp(ext, ".json") == 0) return "application/json";
    if (strcmp(ext, ".xml") == 0) return "application/xml";
    return "application/octet-stream"; // Default MIME type
}
// Send HTTP response
void send_response(SOCKET client_socket, const char *status, const char *content_type, // Function to send HTTP response to the client
                   const char *body, size_t body_len) {
    char header[BUFFER_SIZE];// Buffer to hold the HTTP response header
    
    snprintf(header, sizeof(header),// Format the HTTP response header
             "%sContent-Type: %s\r\n"// Status line and headers
             "Content-Length: %zu\r\n"// Content length header
             "Connection: close\r\n"// Connection header
             "\r\n",
             status, content_type, body_len);
    
    send(client_socket, header, (int)strlen(header), 0);// Send the header to the client
    if (body && body_len > 0) {// If there is a body to send
        send(client_socket, body, (int)body_len, 0);// Send the body to the client
    }
}
// Send 404 Not Found response
void send_404(SOCKET client_socket) { // Function to send a 404 Not Found response
    const char *body = "<html><body><h1>404 Not Found</h1></body></html>";// HTML body for 404 response
    send_response(client_socket, http_404, "text/html", body, strlen(body));// Send the 404 response
}
// Send 500 Internal Server Error response
void send_500(SOCKET client_socket) { // Function to send a 500 Internal Server Error response
    const char *body = "<html><body><h1>500 Internal Server Error</h1></body></html>";// HTML body for 500 response
    send_response(client_socket, http_500, "text/html", body, strlen(body));// Send the 500 response
}
// Serve static file
void serve_file(SOCKET client_socket, const char *filepath) {// Function to serve a static file to the client
    struct _stat st;// Structure to hold file information
    
    // Check if file exists
    if (_stat(filepath, &st) == -1) {// Get file status
        send_404(client_socket);// If file does not exist, send 404 response
        return;
    }
    
    // Open file in binary mode
    FILE *file = fopen(filepath, "rb");// Open the file for reading in binary mode
    if (!file) {    //  If file cannot be opened
        send_500(client_socket);    // Send 500 response
        return;
    }
    
    // Allocate buffer for file content
    char *file_content = (char*)malloc(st.st_size); // Allocate memory for file content
    if (!file_content) {    // If memory allocation fails
        fclose(file);   // Close the file
        send_500(client_socket);    // Send 500 response
        return; 
    }
    
    // Read file
    size_t bytes_read = fread(file_content, 1, st.st_size, file);// Read the file content into the buffer
    fclose(file);// Close the file
    
    if (bytes_read != st.st_size) {//   If the number of bytes read does not match the file size
        free(file_content);     // Free the allocated memory
        send_500(client_socket);        // Send 500 response
        return;
    }
    
    // Send response
    const char *mime_type = get_mime_type(filepath);    // Get the MIME type of the file
    send_response(client_socket, http_200, mime_type, file_content, st.st_size);    // Send the file content as a 200 OK response
    
    free(file_content);     // Free the allocated memory
}
// Handle HTTP request
void handle_request(SOCKET client_socket) {// Function to handle an incoming HTTP request
    char buffer[BUFFER_SIZE];   // Buffer to hold the incoming request data
    int bytes_received = recv(client_socket, buffer, BUFFER_SIZE - 1, 0);   // Receive data from the client
    
    if (bytes_received <= 0) {  // If no data is received or an error occurs
        return;
    }
    
    buffer[bytes_received] = '\0';  // Null-terminate the received data
    
    // Parse request line (GET /path HTTP/1.1)
    char method[16], path[256], protocol[16];   // Buffers to hold the parsed method, path, and protocol
    sscanf(buffer, "%s %s %s", method, path, protocol); // Parse the request line
    
    printf("Request: %s %s %s\n", method, path, protocol);  // Log the request to the console
    
    // Only handle GET requests
    if (strcmp(method, "GET") != 0) {   // If the method is not GET
        const char *body = "<html><body><h1>405 Method Not Allowed</h1></body></html>"; // HTML body for 405 response
        send_response(client_socket, "HTTP/1.1 405 Method Not Allowed\r\n",      // Send 405 response
                     "text/html", body, strlen(body));  
        return;
    }
    
    // Build filepath (serve from current directory)
    char filepath[512] = ".";// Start with current directory
    
    // Default to index.html for root
    if (strcmp(path, "/") == 0) { //    If the requested path is the root
        strcat(filepath, "\\index.html");// Append index.html to the path
    } else {
        // Replace forward slashes with backslashes for Windows
        char *p = path;
        while (*p) {
            if (*p == '/') *p = '\\';
            p++;
        }
        strcat(filepath, path);
    }
    
    // Serve the file
    serve_file(client_socket, filepath);    // Serve the requested file
}
int main() {
    WSADATA wsa_data;
    SOCKET server_socket, client_socket;
    struct sockaddr_in server_addr, client_addr;
    int client_len = sizeof(client_addr);
    
    // Initialize Winsock
    if (WSAStartup(MAKEWORD(2, 2), &wsa_data) != 0) {
        printf("WSAStartup failed. Error Code: %d\n", WSAGetLastError());
        return 1;
    }
    
    printf("Winsock initialized.\n");
    
    // Create socket
    server_socket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (server_socket == INVALID_SOCKET) {
        printf("Socket creation failed. Error Code: %d\n", WSAGetLastError());
        WSACleanup();
        return 1;
    }
    
    // Configure server address
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);
    
    // Bind socket
    if (bind(server_socket, (struct sockaddr*)&server_addr, sizeof(server_addr)) == SOCKET_ERROR) {
        printf("Bind failed. Error Code: %d\n", WSAGetLastError());
        closesocket(server_socket);
        WSACleanup();
        return 1;
    }
    
    // Listen for connections
    if (listen(server_socket, MAX_PENDING) == SOCKET_ERROR) {
        printf("Listen failed. Error Code: %d\n", WSAGetLastError());
        closesocket(server_socket);
        WSACleanup();
        return 1;
    }
    
    printf("Server listening on port %d...\n", PORT);
    printf("Access it at http://localhost:%d\n", PORT);
    printf("Press Ctrl+C to stop the server.\n\n");
    
    // Main server loop
    while (1) {
        client_socket = accept(server_socket, (struct sockaddr*)&client_addr, &client_len);
        if (client_socket == INVALID_SOCKET) {
            printf("Accept failed. Error Code: %d\n", WSAGetLastError());
            continue;
        }
        
        char client_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &(client_addr.sin_addr), client_ip, INET_ADDRSTRLEN);
        printf("Connection from %s:%d\n", client_ip, ntohs(client_addr.sin_port));
        
        handle_request(client_socket);
        closesocket(client_socket);
    }
    
    closesocket(server_socket);
    WSACleanup();
    return 0;
}
