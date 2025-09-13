package main

import (
	"bufio"
	"fmt"
	"net"
	"strings"
	"sync"
)

var (
	clients   = make(map[net.Conn]string)
	clientsMu sync.Mutex
)

func broadcast(sender net.Conn, msg string) {
	clientsMu.Lock()
	defer clientsMu.Unlock()
	for conn := range clients {
		if conn != sender {
			fmt.Fprintln(conn, msg)
		}
	}
}

func handleClient(conn net.Conn) {
	defer conn.Close()

	fmt.Fprint(conn, "Enter your name: ")
	nameInput, _ := bufio.NewReader(conn).ReadString('\n')
	name := strings.TrimSpace(nameInput)

	clientsMu.Lock()
	clients[conn] = name
	clientsMu.Unlock()

	broadcast(conn, fmt.Sprintf("📢 %s has joined the chat", name))

	for {
		msg, err := bufio.NewReader(conn).ReadString('\n')
		if err != nil {
			clientsMu.Lock()
			delete(clients, conn)
			clientsMu.Unlock()
			broadcast(conn, fmt.Sprintf("❌ %s has left the chat", name))
			return
		}
		broadcast(conn, fmt.Sprintf("%s: %s", name, strings.TrimSpace(msg)))
	}
}

func main() {
	listener, _ := net.Listen("tcp", ":8080")
	defer listener.Close()
	fmt.Println("Chat server running on port 8080...")

	for {
		conn, _ := listener.Accept()
		go handleClient(conn)
	}
}
