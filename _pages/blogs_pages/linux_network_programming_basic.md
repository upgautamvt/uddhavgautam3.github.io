# Basic client-server communication using both TCP and UDP.
Here we provide a basic client server communication using both TPC and UDP.

## UDP 
User Datagram Protocol (UDP) is a connectionless protocol unlike connection oriented protocol 
Transmission Control Protocol (TCP).

### Linux UDP socket
```c
//udp_server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUFFER_SIZE 1024

int main(int argc, char *argv[]) {
    //1. process parameters
    if (argc != 4) {
        printf("Syntax: ./server_prog server_ip server_port message_to_send\n");
        return -1;
    }
    char *ip = argv[1];
    int port = atoi(argv[2]); //similar to Integer.parseInt(argv[2]) in Java
    char *message = argv[3];


    //2. create socket
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) {
        perror("Socket creation failed");
        return -1;
    }

    //2. Create sockaddr_in
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr)); //fill with all zeros (i.e., zeroed out)
    //now fill with correct values
    server_addr.sin_family = AF_INET; 
    //TCP-IP uses big-endian. Default in networking is big-endian. 
    //our x86 pc is little-endian. So, any value we receive as big-endian, must be converted to little endian
    //Endianess comes to any multibyte data communication
    server_addr.sin_port = htons(port); //host to network byte order (i.e., big endian)

    // Convert ip address to network binary format and validate IP address
    // return 0 means invalid IP, and -1 means error
    if (inet_pton(AF_INET, ip, &server_addr.sin_addr) <= 0) {
        perror("Invalid address");
        close(sock);
        return -1;
    }

    //4. bind socket
    // we also pass length of server_addr to provide we read full lenght of socket
    if (bind(sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
        perror("Bind failed");
        close(sock);
        return -1;
    }

    printf("Server is listening on %s:%d\n", ip, port);

    //5. create buffer to send/receive message
    // any two buffer to store receive and responseds
    char buffer[BUFFER_SIZE];
    char response[BUFFER_SIZE];
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);


    //6. looper to receive and send
    while (1) {
        memset(buffer, 0, BUFFER_SIZE);
        // in recvfrom function client_addr_len part is a pointer, pointer points to address, therefore, address of the value should be passed. Pass through by reference example
        ssize_t received_len = recvfrom(sock, buffer, BUFFER_SIZE - 1, 0, (struct sockaddr *)&client_addr, &client_addr_len);
        if (received_len < 0) {
            perror("recvfrom failed");
            break;
        }

        buffer[received_len] = '\0'; //null-termination
        printf("Received message from client: %s\n", buffer);

        // Ensure no overflow during message construction
        snprintf(response, BUFFER_SIZE, "%.*s: %.*s", 512, message, 512, buffer);

        ssize_t sent_len = sendto(sock, response, strlen(response), 0, (struct sockaddr *)&client_addr, client_addr_len);
        if (sent_len < 0) {
            perror("sendto failed");
            break;
        }

        printf("Sent response to client: %s\n", response);
    }

    //7. all done, sock can be closed
    close(sock);
    return 0;
}

```
To compile: `gcc -o udp_server udp_server.c`
To start server: `./udp_server 127.0.0.1 8080 "hello"`
To test: `echo "hello server" | nc -u 127.0.0.1 8080`

```shell
!root@amd:/home/upgautam/network_programming_linux# ./udp_server 127.0.0.1 8080 "hello"
Server is listening on 127.0.0.1:8080
Received message from client: hello server
Sent response to client: hello: hello server

# send from client
upgautam@amd:~/network_programming_linux$ echo "hello server" | nc -u 127.0.0.1 8080
hello: hello server
```

Now, we make udp_client.c program to ping to udp_server.c

```c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUFFER_SIZE 1024

int main(int argc, char *argv[]) {
    // Check for correct number of arguments
    if (argc != 4) {
        printf("Usage: %s <server_ip> <server_port> <message>\n", argv[0]);
        return -1;
    }

    // Process command-line arguments
    char *server_ip = argv[1];
    int server_port = atoi(argv[2]);
    char *message = argv[3];

    // Create UDP socket
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) {
        perror("Socket creation failed");
        return -1;
    }

    // Configure server address
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(server_port);

    // Convert server IP to binary format
    if (inet_pton(AF_INET, server_ip, &server_addr.sin_addr) <= 0) {
        perror("Invalid server IP address");
        close(sock);
        return -1;
    }

    // Send message to server
    ssize_t sent_len = sendto(sock, message, strlen(message), 0,
                              (struct sockaddr *)&server_addr, sizeof(server_addr));
    if (sent_len < 0) {
        perror("Failed to send message");
        close(sock);
        return -1;
    }

    printf("Message sent to %s:%d\n", server_ip, server_port);

    // Receive server response
    char buffer[BUFFER_SIZE];
    memset(buffer, 0, BUFFER_SIZE);

    socklen_t server_addr_len = sizeof(server_addr);
    ssize_t received_len = recvfrom(sock, buffer, BUFFER_SIZE - 1, 0,
                                    (struct sockaddr *)&server_addr, &server_addr_len);
    if (received_len < 0) {
        perror("Failed to receive response");
        close(sock);
        return -1;
    }

    // Print server response
    printf("Server response: %s\n", buffer);

    // Close the socket
    close(sock);

    return 0;
}

```

## TCP 
TCP is connection oriented protocol. 

### Linux TCP socket
```c
//tcp_server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <signal.h>

#define BUFFER_SIZE 1024
#define BACKLOG 10 // Number of pending connections allowed in the queue

int server_sock; // Global variable to access and close the socket in signal handler

// Signal handler for SIGINT (Ctrl+C)
void handle_sigint(int sig) {
    printf("\nCaught signal %d. Shutting down server...\n", sig);
    close(server_sock); // Clean up resources
    exit(0); // Exit the program
}

int main(int argc, char *argv[]) {
    //1. process parameters
    if (argc != 4) {
        printf("Syntax: ./server_prog server_ip server_port message_to_send\n");
        return -1;
    }
    char *ip = argv[1];
    int port = atoi(argv[2]); // Convert port number from string to integer
    char *message = argv[3];

    // Set up signal handling for SIGINT
    signal(SIGINT, handle_sigint);

    //2. create socket
    server_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (server_sock < 0) {
        perror("Socket creation failed");
        return -1;
    }

    //3. Create sockaddr_in
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr)); // Zero out the structure

    // Set up server address structure
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    if (inet_pton(AF_INET, ip, &server_addr.sin_addr) <= 0) {
        perror("Invalid IP address");
        close(server_sock);
        return -1;
    }

    //4. bind socket
    if (bind(server_sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Bind failed");
        close(server_sock);
        return -1;
    }

    //4.1. We need to listen in TCP (this we don't do in UDP) 
    // Start listening for incoming connections
    if (listen(server_sock, BACKLOG) < 0) {
        perror("Listen failed");
        close(server_sock);
        return -1;
    }

    printf("Server is listening on %s:%d\n", ip, port);

    //5. create buffer to send/receive message
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    char buffer[BUFFER_SIZE];
    char response[BUFFER_SIZE];

    //6. main looper
    while (1) {
        // 6.1. We also need to accept (this we don't do in UDP)
        // Accept a new connection from a client
        int client_sock = accept(server_sock, (struct sockaddr *)&client_addr, &client_addr_len);
        if (client_sock < 0) {
            perror("Accept failed");
            continue; // Accept failed, try the next connection
        }

        printf("Connected to client\n");

        // Clear the buffer for receiving data
        memset(buffer, 0, BUFFER_SIZE);

        // Receive data from the client
        ssize_t received_len = recv(client_sock, buffer, BUFFER_SIZE - 1, 0);
        if (received_len < 0) {
            perror("Receive failed");
            close(client_sock);
            continue; // Continue with the next client
        }

        buffer[received_len] = '\0'; // Null-terminate the received string
        printf("Received message from client: %s\n", buffer);

        // Construct the response message
        snprintf(response, BUFFER_SIZE, "%s: %s", message, buffer);

        // Send the response back to the client
        ssize_t sent_len = send(client_sock, response, strlen(response), 0);
        if (sent_len < 0) {
            perror("Send failed");
        } else {
            printf("Sent response to client: %s\n", response);
        }

        // Close the connection with the current client
        close(client_sock);
    }

    // This will never be reached unless the loop exits, but is retained for clarity
    close(server_sock);
    return 0;
}

```
To compile: `gcc -o tdp_server tdp_server.c`
To start server: `./tdp_server 127.0.0.1 8080 "hello"`
To test: `echo "hello server" | nc -u 127.0.0.1 8080`

```shell
!root@amd:/home/upgautam/network_programming_linux# ./tcp_server 127.0.0.1 8080 "hello"
Server is listening on 127.0.0.1:8080
Received message from client: hello server
Sent response to client: hello: hello server

# send from client
upgautam@amd:~/network_programming_linux$ echo "hello server" | nc -u 127.0.0.1 8080
hello: hello server
```

Now, we make a tcp_client.c program to ping to tcp_server.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUFFER_SIZE 1024

int main(int argc, char *argv[]) {
    if (argc != 4) {
        printf("Syntax: ./tcp_client server_ip server_port message_to_send\n");
        return -1;
    }

    char *ip = argv[1];
    int port = atoi(argv[2]); // Convert port number from string to integer
    char *message = argv[3];

    // Create a TCP socket
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("Socket creation failed");
        return -1;
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr)); // Zero out the structure

    // Set up server address structure
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);

    // Convert IP address to binary form and validate
    if (inet_pton(AF_INET, ip, &server_addr.sin_addr) <= 0) {
        perror("Invalid IP address");
        close(sock);
        return -1;
    }

    // Connect to the server
    if (connect(sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Connection to the server failed");
        close(sock);
        return -1;
    }

    printf("Connected to the server at %s:%d\n", ip, port);

    // Send the message to the server
    if (send(sock, message, strlen(message), 0) < 0) {
        perror("Message sending failed");
        close(sock);
        return -1;
    }

    printf("Message sent: %s\n", message);

    // Receive the response from the server
    char buffer[BUFFER_SIZE];
    memset(buffer, 0, BUFFER_SIZE);
    ssize_t received_len = recv(sock, buffer, BUFFER_SIZE - 1, 0);
    if (received_len < 0) {
        perror("Response receiving failed");
    } else {
        buffer[received_len] = '\0'; // Null-terminate the response
        printf("Response from server: %s\n", buffer);
    }

    // Close the socket
    close(sock);
    printf("Disconnected from the server\n");
    return 0;
}

```

## Difference between UDP and TCP server program
We have listen and accept functionality in TCP that are used to create connection; that is why connection-oriented.
